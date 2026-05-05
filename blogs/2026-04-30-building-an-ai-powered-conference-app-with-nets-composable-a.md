---
title: "Building an AI-Powered Conference App with .NET’s Composable AI Stack"
url: "https://devblogs.microsoft.com/dotnet/building-ai-conference-app-dotnet-composable-stack/"
date: "Thu, 30 Apr 2026 17:05:00 +0000"
author: "Luis Quintanilla"
feed_url: "https://devblogs.microsoft.com/dotnet/feed/"
---
<p>Building AI features into .NET applications often means stitching together models, vector databases, ingestion pipelines, and agent frameworks from different ecosystems. Each one has its own patterns, its own client libraries, and its own breaking changes when the next version ships. We&#8217;ve been working on a set of composable, extensible building blocks that give you stable abstractions across all of these concerns.</p>
<p>We&#8217;re excited to walk you through how we used them together. For a session at MVP Summit, we built an interactive conference assistant called ConferencePulse. It runs live polls, answers audience questions in real time, generates insights from engagement data, and summarizes the session when it wraps up. We built the app using the exact technologies we were there to present: <code>Microsoft.Extensions.AI</code>, <code>Microsoft.Extensions.DataIngestion</code>, <code>Microsoft.Extensions.VectorData</code>, Model Context Protocol (MCP), and Microsoft Agent Framework.</p>
<p>This post walks through the app and shows how each building block fits.</p>
<h2>What we built</h2>
<p>ConferencePulse is a Blazor Server app for live conference sessions. Attendees scan a QR code, join the session, and interact with the presenter through polls and Q&amp;A. On the backend, AI powers several features:</p>
<ul>
<li><strong>Live polls</strong> that the AI generates based on session content. Attendees vote and results appear in real time.</li>
<li><strong>Audience Q&amp;A</strong> where AI answers questions using a RAG pipeline that pulls from the session knowledge base, Microsoft Learn docs, and GitHub wiki content.</li>
<li><strong>Auto-generated insights</strong> that surface patterns in poll results and audience questions as they come in.</li>
<li><strong>Session summary</strong> that runs when the presenter ends the session. Multiple AI agents analyze polls, questions, and insights concurrently, then merge their findings.</li>
</ul>
<p><img alt="ConferencePulse presenter dashboard showing real-time poll results, audience questions, and AI-generated insights" src="https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2026/04/ai-assistant-presenter-view-scaled.webp" /></p>
<p>We wanted an interactive session, not a slide deck. We wanted polls and audience insights. And we wanted to automate the preparation: point the app at a GitHub repo, and it downloads the markdown, processes it through a pipeline, and builds a searchable knowledge base. Polls, talking points, and Q&amp;A answers are all grounded in that content.</p>
<p>The app runs on .NET 10, Blazor Server, and Aspire. Five projects cover the stack:</p>
<pre><code class="language-text">src/
├── ConferenceAssistant.Web/          ← Blazor Server (UI + orchestration)
├── ConferenceAssistant.Core/         ← Models, interfaces, session state
├── ConferenceAssistant.Ingestion/    ← Data ingestion pipeline + vector search
├── ConferenceAssistant.Agents/       ← AI agents, workflows, tools
├── ConferenceAssistant.Mcp/          ← MCP server tools + MCP client
└── ConferenceAssistant.AppHost/      ← .NET Aspire (Qdrant, PostgreSQL, Azure OpenAI)</code></pre>
<p>Now let&#8217;s walk through the building blocks.</p>
<h2>Microsoft.Extensions.AI: one interface, any provider</h2>
<p><code>Microsoft.Extensions.AI</code> gives you <code>IChatClient</code>, a unified abstraction that works with OpenAI, Azure OpenAI, Ollama, Foundry Local, and other providers. Every AI call in ConferencePulse goes through a single middleware pipeline.</p>
<pre><code class="language-csharp">var openaiBuilder = builder.AddAzureOpenAIClient("openai");

openaiBuilder.AddChatClient("chat")
    .UseFunctionInvocation()
    .UseOpenTelemetry()
    .UseLogging();

openaiBuilder.AddEmbeddingGenerator("embedding");</code></pre>
<p>That&#8217;s it. Six lines. If you&#8217;ve worked with ASP.NET Core middleware, this pattern will feel familiar. Each <code>.Use*()</code> call wraps the inner client with additional behavior. <code>UseFunctionInvocation()</code> handles tool-call loops. <code>UseOpenTelemetry()</code> traces every call. <code>UseLogging()</code> captures request/response pairs.</p>
<p>Want to swap Azure OpenAI for Ollama? Change the inner client. The middleware stays the same.</p>
<p>This matters because <code>IChatClient</code> shows up everywhere in the app. Poll generation, Q&amp;A, insights, ingestion enrichment, and multi-agent workflows all share this pipeline. You register it once and use it throughout.</p>
<h2>DataIngestion + VectorData: the knowledge layer</h2>
<p>AI models need context to give useful answers. <code>Microsoft.Extensions.DataIngestion</code> provides a pipeline for processing documents into searchable chunks. <code>Microsoft.Extensions.VectorData</code> provides a provider-agnostic abstraction over vector stores.</p>
<p>When ConferencePulse imports content from a GitHub repo, it runs the files through an ingestion pipeline:</p>
<pre><code class="language-csharp">IngestionDocumentReader reader = new MarkdownReader();

var tokenizer = TiktokenTokenizer.CreateForModel("gpt-4o");
var chunkerOptions = new IngestionChunkerOptions(tokenizer)
{
    MaxTokensPerChunk = 500,
    OverlapTokens = 50
};
IngestionChunker&lt;string&gt; chunker = new HeaderChunker(chunkerOptions);

var enricherOptions = new EnricherOptions(_chatClient) { LoggerFactory = _loggerFactory };

using var writer = new VectorStoreWriter&lt;string&gt;(
    _searchService.VectorStore,
    dimensionCount: 1536,
    new VectorStoreWriterOptions
    {
        CollectionName = "conference_knowledge",
        IncrementalIngestion = true
    });

using IngestionPipeline&lt;string&gt; pipeline = new(
    reader, chunker, writer, new IngestionPipelineOptions(), _loggerFactory)
{
    ChunkProcessors =
    {
        new SummaryEnricher(enricherOptions),
        new KeywordEnricher(enricherOptions, ReadOnlySpan&lt;string&gt;.Empty),
        frontMatterProcessor
    }
};</code></pre>
<p>The pipeline reads markdown, chunks it by headers, enriches each chunk with AI-generated summaries and keywords, then embeds and stores the results in Qdrant. Each step is a pluggable component. You can swap <code>MarkdownReader</code> for a PDF reader, <code>HeaderChunker</code> for a fixed-size chunker, or Qdrant for Azure AI Search. The pipeline composition stays the same.</p>
<p>Notice that <code>SummaryEnricher</code> and <code>KeywordEnricher</code> both take <code>EnricherOptions(_chatClient)</code>. They use the same <code>IChatClient</code> from the previous section. AI enriching its own context. The summary enricher generates a concise description of each chunk, and the keyword enricher extracts searchable terms. Both improve retrieval quality later.</p>
<p>On the query side, <code>Microsoft.Extensions.VectorData</code> gives you <code>VectorStoreCollection</code> for semantic search over any backend:</p>
<pre><code class="language-csharp">var results = collection.SearchAsync(query, topK);

await foreach (var result in results)
{
    var content = result.Record["content"] as string;
    // Use the content...
}</code></pre>
<p>Similar to how you can swap database providers in EF Core, you can swap vector store providers here. Qdrant today, Azure AI Search tomorrow. Same API.</p>
<p>ConferencePulse also ingests data in real time as the session progresses. Poll responses, audience questions, Q&amp;A pairs, and AI-generated insights all go into the knowledge base:</p>
<pre><code class="language-csharp">public async Task&lt;int&gt; IngestResponseAsync(
    string pollId, string topicId, string question,
    Dictionary&lt;string, int&gt; results, List&lt;string&gt;? otherResponses = null)
{
    var sb = new StringBuilder();
    sb.AppendLine($"Poll: {question}");
    sb.AppendLine("Results:");
    var total = results.Values.Sum();
    foreach (var (option, count) in results)
    {
        var percentage = total &gt; 0 ? (count * 100.0 / total).ToString("F1") : "0";
        sb.AppendLine($"  - {option}: {count} votes ({percentage}%)");
    }

    await _searchService.UpsertAsync(sb.ToString(), source: "response", documentId: $"response-{pollId}");
    return 1;
}</code></pre>
<p>By the end of a session, the knowledge base contains the original imported content, every poll result, every audience question, and every AI-generated insight.</p>
<p><img alt="AI Assistant generating real-time insights from poll results and audience questions" src="https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2026/04/ai-assistant-insights.webp" /></p>
<h2>IChatClient with tools: choosing the right level of complexity</h2>
<p>One of the design principles we followed: use the simplest approach that gets the job done. <code>IChatClient</code> with tools handles a lot of scenarios before you need a dedicated agent framework. At the same time, when orchestration gets complex, a framework earns its place. The key is choosing the right tool.</p>
<p>ConferencePulse has three AI-powered features at different levels of complexity. All three use the same <code>IChatClient</code>.</p>
<h3>Insight generation: a single call</h3>
<p>When a poll closes, ConferencePulse generates an insight. The implementation is a single <code>GetResponseAsync</code> call:</p>
<pre><code class="language-csharp">var response = await chatClient.GetResponseAsync(
[
    new(ChatRole.System,
        "You are a conference analytics assistant generating real-time insights from audience data."),
    new(ChatRole.User, prompt)  // prompt contains the poll results
]);

var content = response.Text?.Trim();
if (!string.IsNullOrWhiteSpace(content))
{
    ctx.AddInsight(new Insight
    {
        TopicId = poll.TopicId,
        PollId = pollId,
        Content = content,
        Type = InsightType.PollAnalysis
    });
}</code></pre>
<p>No tools, no framework. A prompt with poll results as context, and the middleware pipeline handles telemetry and logging.</p>
<h3>Poll generation: IChatClient with tools</h3>
<p>Generating a poll needs more context. The AI checks the current topic, looks at what&#8217;s been covered, and creates something relevant. That means tools:</p>
<pre><code class="language-csharp">public class PollGenerationWorkflow(IChatClient chatClient, AgentTools tools)
{
    public async Task&lt;string&gt; ExecuteAsync(string topicId)
    {
        var options = new ChatOptions
        {
            Tools = [tools.GetCurrentTopic, tools.SearchKnowledge,
                     tools.GetAudienceQuestions, tools.GetAllPollResults,
                     tools.GetAllInsights, tools.CreatePoll]
        };

        var messages = new List&lt;ChatMessage&gt;
        {
            new(ChatRole.System, AgentDefinitions.SurveyArchitectInstructions),
            new(ChatRole.User, $"Generate an engaging poll for topic: {topicId}...")
        };

        var response = await chatClient.GetResponseAsync(messages, options);
        return response.Text ?? "Unable to generate poll.";
    }
}</code></pre>
<p>Each tool is a strongly-typed <code>AITool</code> property created from a C# method:</p>
<pre><code class="language-csharp">public class AgentTools
{
    public AITool SearchKnowledge { get; }
    public AITool GetCurrentTopic { get; }
    public AITool CreatePoll { get; }
    // ...

    public AgentTools(IPollService pollService, ISemanticSearchService searchService, ...)
    {
        SearchKnowledge = AIFunctionFactory.Create(SearchKnowledgeCore,
            new AIFunctionFactoryOptions
            {
                Name = nameof(SearchKnowledge),
                Description = "Search the session knowledge base for content related to the query"
            });
        // ...
    }
}</code></pre>
<p>The model decides it needs context, calls <code>GetCurrentTopic</code> and <code>SearchKnowledge</code>, then generates a poll and calls <code>CreatePoll</code> to save it. The <code>UseFunctionInvocation()</code> middleware handles the tool loop automatically.</p>
<p><img alt="AI assistant conducting a poll in the conference room view" src="https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2026/04/ai-assistant-poll-room-view-scaled.webp" /></p>
<h3>Q&amp;A answering: RAG across multiple sources</h3>
<p>The Q&amp;A service brings multiple building blocks together. When an audience member asks a question, the app searches the local knowledge base, queries Microsoft Learn docs via MCP, and asks DeepWiki about relevant GitHub repos via MCP. Then it synthesizes an answer:</p>
<pre><code class="language-csharp">// 1. Search local knowledge base
var searchResults = await searchService.SearchAsync(questionText, topK: 5);
var localContext = string.Join("\n\n---\n\n",
    searchResults.Select(r =&gt; r.Content).Where(c =&gt; !string.IsNullOrWhiteSpace(c)));

// 2. Search Microsoft Learn for documentation context (via MCP)
var docsContext = await mcpClient.SearchDocsAsync(questionText);

// 3. Ask DeepWiki about relevant .NET repos (via MCP)
var deepWikiContext = await mcpClient.AskDeepWikiAsync("dotnet/extensions", questionText);</code></pre>
<p>VectorData for local search, MCP for external context, <code>IChatClient</code> for generation.</p>
<p><!-- Image of an AI assistant QA interface showing conversational interaction with an artificial intelligence system --></p>
<p><img alt="AI Assistant QA Interface" src="https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2026/04/ai-assistant-qa.webp" /></p>
<p>Now let&#8217;s look at how MCP works.</p>
<h2>MCP: consuming and providing context</h2>
<p>Model Context Protocol is a standard for AI applications to discover and use external tools and context. Similar to how HTTP lets any client talk to any server, MCP lets any AI app connect to any context provider using the same protocol.</p>
<p>ConferencePulse uses MCP in both directions.</p>
<h3>As a consumer</h3>
<p>The <code>McpContentClient</code> connects to two MCP servers at startup: Microsoft Learn and DeepWiki.</p>
<pre><code class="language-csharp">public async Task InitializeAsync(CancellationToken ct = default)
{
    var learnTransport = new HttpClientTransport(new HttpClientTransportOptions
    {
        Endpoint = new Uri("https://learn.microsoft.com/api/mcp"),
        TransportMode = HttpTransportMode.StreamableHttp
    }, loggerFactory);
    _learnClient = await McpClient.CreateAsync(learnTransport, null, loggerFactory, ct);

    var deepWikiTransport = new HttpClientTransport(new HttpClientTransportOptions
    {
        Endpoint = new Uri("https://mcp.deepwiki.com/mcp"),
        TransportMode = HttpTransportMode.StreamableHttp
    }, loggerFactory);
    _deepWikiClient = await McpClient.CreateAsync(deepWikiTransport, null, loggerFactory, ct);
}</code></pre>
<p>Once connected, calling a tool on any MCP server uses the same pattern:</p>
<pre><code class="language-csharp">var result = await _learnClient.CallToolAsync(
    "microsoft_docs_search",
    new Dictionary&lt;string, object?&gt; { ["query"] = query },
    cancellationToken: ct);</code></pre>
<p>Any server that speaks MCP works with this client code.</p>
<h3>As a provider</h3>
<p>ConferencePulse is also an MCP server. Any MCP-compatible client (GitHub Copilot, Claude, a custom tool) can connect and query session data.</p>
<pre><code class="language-csharp">[McpServerToolType]
public class ConferenceTools
{
    [McpServerTool(Name = "get_session_status", ReadOnly = true),
     Description("Returns the current conference session status.")]
    public static string GetSessionStatus(ISessionService sessionService)
    {
        var session = sessionService.CurrentSession;
        if (session is null) return "No active conference session.";
        // ... build status string
    }

    [McpServerTool(Name = "search_session_knowledge", ReadOnly = true),
     Description("Searches the session knowledge base for relevant content.")]
    public static async Task&lt;string&gt; SearchSessionKnowledge(
        ISemanticSearchService searchService,
        [Description("The search query.")] string query,
        [Description("Max results. Defaults to 5.")] int maxResults = 5)
    {
        var results = await searchService.SearchAsync(query, maxResults);
        // ... format results
    }
}</code></pre>
<p>Registration takes a few lines in <code>Program.cs</code>:</p>
<pre><code class="language-csharp">builder.Services
    .AddMcpServer(options =&gt; { options.ServerInfo = new() { Name = "ConferencePulse", Version = "1.0.0" }; })
    .WithToolsFromAssembly(typeof(ConferenceTools).Assembly)
    .WithHttpTransport();

app.MapMcp("/mcp");</code></pre>
<p>The app consumes external knowledge to answer questions and provides its own data for external tools. Same protocol in both directions.</p>
<h2>Microsoft Agent Framework: multi-agent orchestration</h2>
<p>For most of ConferencePulse&#8217;s features, <code>IChatClient</code> with tools was the right choice. But the session summary needed something more: three specialized agents running concurrently, each with scoped tools, feeding their results into a synthesis step. That&#8217;s where Microsoft Agent Framework comes in.</p>
<pre><code class="language-csharp">public class SessionSummaryWorkflow(IChatClient chatClient, AgentTools tools)
{
    public async Task&lt;string&gt; ExecuteAsync()
    {
        ChatClientAgent pollAnalyst = new(chatClient,
            name: "PollAnalyst",
            description: "Analyzes poll results and trends",
            instructions: "You are a poll analyst. Use GetAllPollResults to retrieve every poll...",
            tools: [tools.GetAllPollResults]);

        ChatClientAgent questionAnalyst = new(chatClient,
            name: "QuestionAnalyst",
            description: "Analyzes audience questions and themes",
            instructions: "You are an audience question analyst...",
            tools: [tools.GetAudienceQuestions]);

        ChatClientAgent insightAnalyst = new(chatClient,
            name: "InsightAnalyst",
            description: "Analyzes generated insights and knowledge patterns",
            instructions: "You are an insight analyst...",
            tools: [tools.GetAllInsights, tools.SearchKnowledge]);</code></pre>
<p>Each <code>ChatClientAgent</code> wraps the same <code>IChatClient</code>. The agents get scoped tools (PollAnalyst only sees poll data, QuestionAnalyst only sees questions) and specialized instructions.</p>
<p>The orchestration uses <code>AgentWorkflowBuilder.BuildConcurrent</code> for the fan-out, then <code>WorkflowBuilder</code> to compose the full pipeline:</p>
<pre><code class="language-csharp">        // Fan-out: three analysts run concurrently
        var analysisWorkflow = AgentWorkflowBuilder.BuildConcurrent(
            [pollAnalyst, questionAnalyst, insightAnalyst],
            MergeAgentOutputs);

        // Fan-in: synthesizer merges all findings
        ChatClientAgent synthesizer = new(chatClient,
            name: "Synthesizer",
            instructions: "Synthesize the analyses into one cohesive session summary...");

        // Compose: concurrent analysis → sequential synthesis
        var analysisExec = new SubworkflowBinding(analysisWorkflow, "Analysis");
        ExecutorBinding synthExec = synthesizer;

        var composedWorkflow = new WorkflowBuilder(analysisExec)
            .WithName("SessionSummaryPipeline")
            .BindExecutor(synthExec)
            .AddEdge(analysisExec, synthExec)
            .WithOutputFrom([synthExec])
            .Build();

        var run = await InProcessExecution.Default.RunAsync(
            composedWorkflow,
            "Analyze the conference session data and provide your specialized findings.");</code></pre>
<p>Compare this with the poll generation workflow from earlier, which is about 10 lines using <code>IChatClient</code> and tools. The session summary is about 40 lines because it genuinely needs concurrent agents with scoped tools and a synthesis step.</p>
<p>In ConferencePulse, the Agent Framework was the right choice for exactly one workflow. Everything else worked well with <code>IChatClient</code> directly. Both approaches use the same underlying abstraction.</p>
<h2>How the building blocks fit together</h2>
<p><img alt="Aspire Dashboard showing ConferencePulse services: web app, Qdrant, PostgreSQL, and Azure OpenAI" src="https://devblogs.microsoft.com/dotnet/wp-content/uploads/sites/10/2026/04/ai-assistant-aspire-dashboard-scaled.webp" /></p>
<p>During the MVP Summit session, attendees interacted with features powered by different layers of the stack:</p>
<table>
<thead>
<tr>
<th>Feature</th>
<th>Powered by</th>
</tr>
</thead>
<tbody>
<tr>
<td>Polls</td>
<td><code>IChatClient</code> + tools (MEAI)</td>
</tr>
<tr>
<td>Knowledge grounding</td>
<td><code>IngestionPipeline</code> + <code>VectorStoreWriter</code></td>
</tr>
<tr>
<td>Q&amp;A answers</td>
<td><code>VectorData</code> + <code>IChatClient</code> + MCP</td>
</tr>
<tr>
<td>Auto-generated insights</td>
<td><code>IChatClient</code> (single call)</td>
</tr>
<tr>
<td>Session summary</td>
<td>Microsoft Agent Framework (fan-out/fan-in)</td>
</tr>
<tr>
<td>Observability</td>
<td><code>UseOpenTelemetry()</code> + Aspire Dashboard</td>
</tr>
<tr>
<td>Infrastructure</td>
<td>Aspire: Qdrant + PostgreSQL + Azure OpenAI</td>
</tr>
</tbody>
</table>
<p>Each building block handles one concern and composes with the others. <code>IChatClient</code> shows up inside the ingestion enrichers, inside the agent tools, inside the MCP-augmented Q&amp;A, and inside the Agent Framework&#8217;s <code>ChatClientAgent</code>. You learn it once and use it everywhere.</p>
<p>Providers will change and models will evolve. The building blocks give you a stable layer to build on, and you swap implementations underneath without rewriting application code.</p>
<h2>Get started</h2>
<p>We&#8217;re excited to see what you build with these building blocks.</p>
<ul>
<li><strong>Try ConferencePulse</strong>: <a href="https://github.com/luisquintanilla/dotnet-ai-conference-assistant">the source is on GitHub</a>. Clone it, run <code>aspire run</code>, and see the full stack in action.</li>
<li><strong>Learn more</strong> about the individual libraries:
<ul>
<li><a href="https://learn.microsoft.com/dotnet/ai/ai-extensions">Microsoft.Extensions.AI</a></li>
<li><a href="https://learn.microsoft.com/dotnet/ai/vector-stores/overview">Microsoft.Extensions.VectorData</a></li>
<li><a href="https://learn.microsoft.com/dotnet/ai/conceptual/data-ingestion">Microsoft.Extensions.DataIngestion</a></li>
<li><a href="https://learn.microsoft.com/dotnet/ai/get-started-mcp">Model Context Protocol in .NET</a></li>
<li><a href="https://github.com/microsoft/agent-framework">Microsoft Agent Framework</a></li>
</ul>
</li>
<li><strong>Give us feedback</strong>: file an issue in any of the repos or catch us on the <a href="https://dotnet.microsoft.com/live">.NET Community Standup</a>.</li>
</ul>
<p>Now that you&#8217;ve seen how these building blocks compose, give them a try and let us know what you think.</p>
<p>The post <a href="https://devblogs.microsoft.com/dotnet/building-ai-conference-app-dotnet-composable-stack/">Building an AI-Powered Conference App with .NET&#8217;s Composable AI Stack</a> appeared first on <a href="https://devblogs.microsoft.com/dotnet">.NET Blog</a>.</p>
