# Nemotron Synthetic Data Generation Reference

Sources:
- [Synthetic Data Generation](https://docs.nvidia.com/nemo-framework/user-guide/24.12/datacuration/syntheticdata.html)
- [NeMo Curator](https://docs.nvidia.com/nemo-framework/user-guide/latest/)

## Overview

NeMo Curator provides synthetic data generation using Nemotron models for creating training data. Nemotron-4 340B used "98% synthetic data for supervised fine-tuning and preference fine-tuning."

## Setup

### Client Configuration

```python
from nemo_curator.synthetic import OpenAIClient, NemotronGenerator

# NVIDIA Build API
client = OpenAIClient(
    base_url="https://integrate.api.nvidia.com/v1",
    api_key="your_nvidia_api_key",
    model_name="nvidia/nemotron-3-nano-30b-a3b"
)

# Self-hosted NIM
client = OpenAIClient(
    base_url="http://localhost:8000/v1",
    api_key="not-used",
    model_name="nvidia/nemotron-3-nano-30b-a3b"
)

# Initialize generator
generator = NemotronGenerator(client)
```

### Self-Hosted with Conversation Formatter

```python
from nemo_curator.synthetic import NeMoDeployClient
from nemo_curator.synthetic.conversation import Mixtral8x7BFormatter

# NeMo Deploy (local inference)
client = NeMoDeployClient(
    base_url="http://localhost:8000",
    model_name="nemotron-3-nano",
    conversation_formatter=Mixtral8x7BFormatter()
)

generator = NemotronGenerator(client)
```

## Data Generation Pipelines

### Open Q&A Pipeline

```python
# Generate open-domain Q&A
df = generator.run_open_qa_pipeline(
    n_macro_topics=10,          # Number of macro topics
    n_subtopics=20,             # Subtopics per macro topic
    n_openlines=5,              # Questions per subtopic
    output_file="openqa.jsonl"
)

# Process:
# 1. Generate macro topics
# 2. Generate subtopics for each macro topic
# 3. Generate open-ended questions
# 4. Revise questions for quality
```

### Writing Pipeline

```python
# Generate writing prompts
df = generator.run_writing_pipeline(
    n_macro_topics=5,
    n_subtopics=10,
    output_file="writing.jsonl"
)

# Generates prompts for:
# - Creative writing
# - Technical writing
# - Academic writing
# - Business writing
```

### Math Pipeline

```python
# Generate math problems
df = generator.run_math_pipeline(
    n_macro_topics=4,
    n_subtopics=8,
    school_level="high school",  # beginner, middle school, high school, university
    output_file="math.jsonl"
)

# School levels:
# - beginner: Basic arithmetic
# - middle school: Pre-algebra, geometry
# - high school: Algebra, calculus
# - university: Advanced mathematics
```

### Python Coding Pipeline

```python
# Generate Python problems
df = generator.run_python_pipeline(
    n_macro_topics=6,
    n_subtopics=12,
    output_file="python.jsonl"
)

# Generates:
# - Algorithm problems
# - Data structure challenges
# - System design questions
# - Debugging exercises
```

## Manual Generation Methods

### Macro Topics

```python
# Generate high-level topics
topics = generator.generate_macro_topics(
    n_macro_topics=10,
    prompt_kwargs={"domain": "machine learning"}
)

# Returns list of topics
# ['Supervised Learning', 'Neural Networks', ...]
```

### Subtopics

```python
# Generate subtopics for a topic
subtopics = generator.generate_subtopics(
    macro_topic="Neural Networks",
    n_subtopics=5
)

# Returns list of subtopics
# ['Convolutional Networks', 'RNNs', 'Transformers', ...]
```

### Questions from Topics

```python
# Open Q&A questions
questions = generator.generate_openlines_from_topic(
    topic="Transformers",
    n_openlines=3
)

# Math problems
math_problems = generator.generate_math_from_topic(
    macro_topic="Calculus",
    subtopic="Derivatives",
    school_level="high school",
    n_samples=5
)

# Python problems
python_problems = generator.generate_python_from_topic(
    macro_topic="Data Structures",
    subtopic="Binary Trees",
    n_samples=5
)
```

## Dialogue Generation

### Multi-Turn Conversations

```python
# Generate multi-turn dialogue
dialogue = generator.generate_dialogue(
    persona="expert Python programmer",
    n_user_turns=3,
    topic="async programming",
    context="Teaching asyncio to beginners"
)

# Returns structured conversation:
# [
#   {"role": "user", "content": "..."},
#   {"role": "assistant", "content": "..."},
#   {"role": "user", "content": "..."},
#   ...
# ]
```

### Two-Turn Prompts

```python
# Generate initial prompt + follow-up
two_turn = generator.generate_two_turn_prompt(
    topic="machine learning",
    n_samples=100
)

# Format:
# User: Initial question
# Assistant: Response
# User: Follow-up question
```

## Customization

### Custom Prompt Templates

```python
# Override default templates
custom_template = """
Generate {n_samples} questions about {topic}.
Focus on {domain} applications.
"""

questions = generator.generate_openlines_from_topic(
    topic="Neural Networks",
    n_openlines=5,
    prompt_template=custom_template,
    prompt_kwargs={"domain": "computer vision"}
)
```

### Generation Parameters

```python
# Control model behavior
generator = NemotronGenerator(
    client,
    temperature=0.8,        # Creativity (0.0-1.0)
    top_p=0.95,            # Nucleus sampling
    max_tokens=2048,       # Max response length
    frequency_penalty=0.1, # Reduce repetition
    presence_penalty=0.1   # Encourage diversity
)
```

## Asynchronous Generation

### Concurrent Requests

```python
from nemo_curator.synthetic import AsyncNemotronGenerator

# Async generator with rate limiting
async_gen = AsyncNemotronGenerator(
    client,
    max_concurrent_requests=10  # Concurrent API calls
)

# Large-scale generation
df = async_gen.run_open_qa_pipeline(
    n_macro_topics=100,
    n_subtopics=200,
    output_file="large_dataset.jsonl"
)

# Automatically handles:
# - Concurrent requests
# - Rate limiting
# - Error retry
# - Progress tracking
```

## Response Processing

### YAML Conversion

```python
# Convert LLM response to list
from nemo_curator.synthetic import convert_response_to_yaml_list

response = "- Question 1\n- Question 2\n- Question 3"
questions = convert_response_to_yaml_list(
    response,
    ignore_conversion_failure=True  # Continue on parse errors
)

# Returns: ['Question 1', 'Question 2', 'Question 3']
```

## Integration with NeMo Curator

### Full Pipeline

```python
from nemo_curator import DocumentDataset
from nemo_curator.filters import WordCountFilter
from nemo_curator.modules import ExactDuplicates

# 1. Generate synthetic data
generator = NemotronGenerator(client)
df = generator.run_open_qa_pipeline(
    n_macro_topics=10,
    n_subtopics=20
)

# 2. Convert to DocumentDataset
dataset = DocumentDataset.from_pandas(df)

# 3. Filter by word count
filtered = dataset.filter(WordCountFilter(min_words=10, max_words=500))

# 4. Deduplicate
deduped = ExactDuplicates().deduplicate(filtered)

# 5. Export
final_df = deduped.to_pandas()
final_df.to_json("curated_data.jsonl", orient="records", lines=True)
```

### Quality Filtering

```python
from nemo_curator.filters import QualityFilter

# Apply quality filters
quality_filtered = dataset.filter(
    QualityFilter(
        min_word_count=50,
        max_repetition_ratio=0.2,
        min_unique_words_ratio=0.3
    )
)
```

## Reward Model Scoring

### Conversation Scoring

```python
# Score generated dialogues
scores = client.query_reward_model(
    conversations=[
        [
            {"role": "user", "content": "What is AI?"},
            {"role": "assistant", "content": "AI is..."}
        ]
    ]
)

# Returns scores for:
# - helpfulness
# - correctness
# - coherence
# - complexity
# - verbosity

# Filter by score
high_quality = [conv for conv, score in zip(conversations, scores) 
                if score['helpfulness'] > 0.8]
```

## Best Practices

### Data Generation

1. **Start small**: Test with small batches first
2. **Iterate**: Refine prompts based on output quality
3. **Diversity**: Use multiple topics and subtopics
4. **Balance**: Mix different pipeline types
5. **Validation**: Manually review samples

### Performance

1. **Async generation**: Use for large-scale production
2. **Rate limiting**: Respect API limits
3. **Caching**: Store intermediate results
4. **Batch size**: Optimize concurrent requests
5. **Error handling**: Implement retry logic

### Quality

1. **Prompt engineering**: Customize templates for domain
2. **Filtering**: Remove low-quality samples
3. **Deduplication**: Avoid repetitive data
4. **Scoring**: Use reward models for ranking
5. **Human review**: Sample-based validation

## Common Patterns

### Supervised Fine-Tuning (SFT) Data

```python
# Generate instruction-following data
sft_data = []

# Open Q&A
qa = generator.run_open_qa_pipeline(n_macro_topics=20)
sft_data.append(qa)

# Writing tasks
writing = generator.run_writing_pipeline(n_macro_topics=10)
sft_data.append(writing)

# Math problems
math = generator.run_math_pipeline(n_macro_topics=5, school_level="high school")
sft_data.append(math)

# Coding problems
code = generator.run_python_pipeline(n_macro_topics=10)
sft_data.append(code)

# Combine and deduplicate
import pandas as pd
combined = pd.concat(sft_data)
deduped = ExactDuplicates().deduplicate(DocumentDataset.from_pandas(combined))
```

### Preference Data

```python
# Generate paired responses
topics = generator.generate_macro_topics(n_macro_topics=10)

preference_data = []
for topic in topics:
    questions = generator.generate_openlines_from_topic(topic, n_openlines=5)
    
    for question in questions:
        # Generate multiple responses
        response1 = client.query_model(question, temperature=0.7)
        response2 = client.query_model(question, temperature=0.9)
        
        # Score responses
        score1 = client.query_reward_model([[{"role": "user", "content": question},
                                             {"role": "assistant", "content": response1}]])
        score2 = client.query_reward_model([[{"role": "user", "content": question},
                                             {"role": "assistant", "content": response2}]])
        
        # Choose preferred
        preferred = response1 if score1['helpfulness'] > score2['helpfulness'] else response2
        rejected = response2 if score1['helpfulness'] > score2['helpfulness'] else response1
        
        preference_data.append({
            "prompt": question,
            "chosen": preferred,
            "rejected": rejected
        })
```

### Domain-Specific Generation

```python
# Medical domain
medical_gen = NemotronGenerator(
    client,
    temperature=0.5  # Lower for factual accuracy
)

medical_data = medical_gen.run_open_qa_pipeline(
    n_macro_topics=10,
    prompt_kwargs={
        "domain": "medical sciences",
        "focus": "clinical diagnosis"
    }
)

# Legal domain
legal_data = medical_gen.run_writing_pipeline(
    n_macro_topics=5,
    prompt_kwargs={
        "domain": "legal",
        "style": "formal"
    }
)
```
