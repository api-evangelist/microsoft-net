---
title: "Durable Workflows in the Microsoft Agent Framework"
url: "https://devblogs.microsoft.com/dotnet/durable-workflows-in-microsoft-agent-framework/"
date: "2026-05-06"
author: "Shyju Krishnankutty"
feed_url: "https://devblogs.microsoft.com/dotnet/feed/"
---
The Microsoft Agent Framework (MAF) is an open-source, multi-language framework for building, orchestrating, and deploying AI agents. Since its preview announcement, the framework has added a workflow programming model that lets you compose multiple agents and other units of work into multi-step pipelines. You define individual steps called executors, wire them into a directed graph using a workflow builder, and the framework handles execution, data flow between steps, and error propagation.
