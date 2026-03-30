+++
date = '2026-03-30T12:11:53+02:00'
title = 'Multi Agent System Crew'
[params]
    description = 'Some notes while learning about CrewAI framework. Save your time and go read the official documentation.'
+++

# CrewAI



```
$ uv tool install crewai --upgrade
```

Available providers:
1. openai
2. anthropic
3. gemini
4. nvidia_nim
5. groq
6. huggingface
7. ollama
8. watson
9. bedrock
10. azure
11. cerebras
12. sambanova

I will use gemini provider:

```sh
$ uv add "crewai[google-genai]"
```

File: .env
```sh
MODEL=gemini/gemini-2.5-flash
GEMINI_API_KEY=AIz...
```

Get the API key here: https://aistudio.google.com/api-keys

## Crew

Docs: https://docs.crewai.com/en/concepts/crews

A crew in crewAI represents a collaborative group of agents working together to achieve a set of tasks. Each crew defines the strategy for task execution, agent collaboration, and the overall workflow.

```py
crew = Example().crew()
crew.kickoff(inputs={'topic': 'AI Agents'})
```

### Agents

Docs: https://docs.crewai.com/en/concepts/agents

An agent is an autonomous unit that can:
* Perform specific tasks
* Make decisions based on its role and goal
* Use tools to accomplish objectives
* Communicate and collaborate with other agents
* Maintain memory of interactions
* Delegate tasks when allowed

There are two ways to create agents in CrewAI: using YAML configuration (recommended) or defining them directly in code.

```yaml
researcher:
  role: >
    {topic} Senior Data Researcher
  goal: >
    Uncover cutting-edge developments in {topic}
  backstory: >
    You're a seasoned researcher with a knack for uncovering the latest
    developments in {topic}. Known for your ability to find the most relevant
    information and present it in a clear and concise manner.
```

Variables in your YAML files (e.g.: `{topic}`) will be replaced with values then inputs dict when running the crew:

```py
@CrewBase
class Example():
    """Example crew"""
  
    @agent
    def researcher(self) -> Agent:
        return Agent(
            config=self.agents_config['researcher'], # type: ignore[index]
            verbose=True,
            tools=[SerperDevTool()]
        )
    
    @crew
    def crew(self) -> Crew:
        """Creates the Example crew"""

        return Crew(
            agents=self.agents, # Automatically created by the @agent decorator
            tasks=self.tasks, # Automatically created by the @task decorator
            process=Process.sequential,
            verbose=True,
        )
```

### Tasks

Docs: https://docs.crewai.com/en/concepts/tasks

```py
from crewai import Task

task = Task(
    description="Evaluate the quality of the given text and provide recommendations.",
    expected_output="A score and a list of recommendations.",
    agent=agent,
    output_pydantic=AgentOutput
)
```

## Flows

Docs: https://docs.crewai.com/en/concepts/flows

### Human-in-the-Loop

The `@human_feedback` decorator enables human-in-the-loop workflows by pausing flow execution to collect feedback from a human. This is useful for approval gates, quality review, and decision points that require human judgment.

## Skills

Docs: https://docs.crewai.com/en/concepts/skills

Skills are self-contained directories that provide agents with domain-specific instructions, references, and assets. Each skill is defined by a SKILL.md file with YAML frontmatter and a markdown body.

```md
---
name: my-skill
description: Short description of what this skill does and when to use it.
license: Apache-2.0                    # optional
compatibility: crewai>=0.1.0           # optional
metadata:                              # optional
  author: your-name
  version: "1.0"
allowed-tools: web-search file-read    # optional, space-delimited
---

Instructions for the agent go here. This markdown body is injected
into the agent's prompt when the skill is activated.
```

## Tools

Docs: https://docs.crewai.com/en/concepts/tools

```sh
$ pip install 'crewai[tools]'
```

```py
from crewai_tools import (
    DirectoryReadTool,
    FileReadTool,
    SerperDevTool,
    WebsiteSearchTool
)
```

### Custom tools

```py
from crewai.tools import BaseTool
from typing import Type
from pydantic import BaseModel, Field


class MyCustomToolInput(BaseModel):
    """Input schema for MyCustomTool."""
    argument: str = Field(..., description="Description of the argument.")

class MyCustomTool(BaseTool):
    name: str = "Name of my tool"
    description: str = (
        "Clear description for what this tool is useful for, your agent will need this information to use it."
    )
    args_schema: Type[BaseModel] = MyCustomToolInput

    def _run(self, argument: str) -> str:
        # Implementation goes here
        return "this is an example of a tool output, ignore it and move along."
```

# References

* https://docs.crewai.com/
* https://github.com/crewAIInc/crewAI
* https://github.com/joaomdmoura/crewai-tools
* https://github.com/langchain-ai/langchain-community/tree/main/libs/community/langchain_community/tools
* https://docs.pydantic.dev/latest/concepts/models/
