# D3-Agentic-Prompting-framework

*Important note: I suggest that, the other repositories that include the rest of the models are studied first, in order to understand the comparisons made here.

## Executive Summary

This report tracks the path of a response clarity classifier designed to handle political Question-
Answering pairs from the ailsntua/QEvasion (CLARITY) dataset. The objective is to put
responses into three distinct categories: Clear Reply, Ambivalent, and Clear Non-Reply. Four
fundamentally unique paradigms were deployed:
1. TF-IDF and Word2Vec models combined with Logistic Regression.
2. In-context learning using the Qwen3.5 family (0.8B, 2B, 4B) across Zero-Shot, Few-Shot,
and Chain-of-Thought (CoT) setups.
3. Encoder-only Transformers (BERT, DistilBERT, DeBERTa-v3) .
4. The D3-Agentic Prompting framework, dividing reasoning among 4 subtask-specific agents
powered by Qwen3.5-0.8B-Instruct.

## Key Paradigm Insight

The performance we see across these assignments are not the ones we would immidietly ex-
pect. Bigger does not mean better and more complex is not always more accurate. The
top-performing system across the entire benchmark is DistilBERT-base-uncased (Macro F1 =
0.62), achieved through fine-tuning and feature engineering. This approach outpaced both the
best Qwen3.5 family configuration (Qwen3.5-2B CoT, Macro F1 = 0.60) and the D3 agent
(D3-Agentic Qwen3.5-0.8B, Macro F1 = 0.41). We understand that when dealing with highly
localized class boundaries, fine-tuning of a smaller encoder can outperform the zero-shot or
multi-agent reasoning structures of auto-regressive models.

# Methodology & Architecture Across Assignments

## TF-IDF and Word2Vec models
TF-IDF + Logistic Regression (C = 1): Built on a high-dimensional feature space limited to
100,000 features. Custom class weights were introduced to offset major imbalances.
Word2Vec + Logistic Regression: Generated document vectors by averaging word embeddings.
This approach discarded word ordering and phrase structure, resulting in significant information
loss. Custom class weights did not seem to habe an effect.

## Qwen3.5 family (0.8B, 2B, 4B)
Made under three model scales (Qwen3.5-0.8B, Qwen3.5-2B, Qwen3.5-4B) and under three
prompting strategies:
Zero-Shot: Direct classification prompt.
Few-Shot: Provide a single representative question-answer example drawn from the training
dataset.
Chain-of-Thought (CoT): Give QA pair with an explicit, step-by-step reasoning path to guide
the model’s analytical process before outputting the final label.

## Encoder-only Transformers
This paradigm fine-tuned encoder models natively on the dataset. The question and answer
fields were joined using a separator token ([SEP]).
Feature Engineering: The data analysis showed that Ambivalent answers were structurally the
longest on average, while Clear Non-Reply instances were the smallest. To take advantage of
this pattern, a custom architecture named LengthAwareClassifier was built..
Models Trained: BERT-base-uncased, DistilBERT-base-uncased, and DeBERTa-v3-base. Opti-
mization relied on AdamW, a linear learning rate scheduler with a 10% warm-up phase.

## D3-Agentic
Question Intent Agent: Passes the question to establish a baseline of what a direct response
must have (question type, question intent).
Answer Content Agent: Summarizes and extracts clear claims or evasive signals (direct answer
present, off target content).
Gap and Evasion Agent: Compares intermediate outputs from the first two agents to classify
specific evasion tactics (dodging, deflection, implicit answer).
Decision Agent: Adds the reasoning steps into a structured JSON payload containing the final
clarity classification label.

## Comprehensive Performance Comparison

The next table shows the performance metrics compiled across all four assignments.
Model                   Accuracy   Macro F1-Score
TF-IDF                    0.60         0.57
Word2Vec                  0.55         0.48
BERT-base-uncased         0.62         0.58
DistilBERT-base-uncased   0.60         0.62
DeBERTa-v3-base           0.59         0.25
Qwen3.5-0.8B (Zero-Shot)   —           0.55
Qwen3.5-0.8B (Few-Shot)    —           0.57
Qwen3.5-0.8B (CoT)         —           0.57
Qwen3.5-2B (Zero-Shot)     —           0.12
Qwen3.5-2B (Few-Shot)      —           0.30
Qwen3.5-2B (CoT)           —           0.60
Qwen3.5-4B (Zero-Shot)     —           0.10
Qwen3.5-4B (Few-Shot)      —           0.43
Qwen3.5-4B (CoT)           —           0.10
D3-Agentic (Qwen3.5-0.8B) 0.54         0.41

# Error Analysis

## Systemic Breakdown
The Clear Non-Reply class remains a primary point of failure across nearly all models. Being
10% of the dataset, it becomes a difficultie for classifiers to overcome.
Linguistic Confusion: TF-IDF and Word2Vec models frequently misclassified Clear Non-Reply
responses as Clear Reply. This is a direct result of a surface level bias since both categories
often employ assertive vocabulary, direct syntax, and formal language. The models mistake this
direct presentation style for factual responsiveness, ignoring the actual meaning.
Cascading Error Propagation: In the D3-Agentic framework, the Clear Non-Reply F1 score
dropped significantly to 0.13. Because the task is broken down sequentially, if the Answer
Content Agent or Gap Agent misses subtle evasion clues, that incomplete context is passed
forward. The final Decision Agent then acts on wrong inputs.

## Structural and Training Optimization Collapse
It is clear to see the catastrophic training failures of the largest generative model and the most
advanced encoder model:
Qwen3.5-4B Format Failure: Scaling up to the 4B parameter model caused performance to drop
during Zero-Shot and CoT inference (0.10 Macro F1). Rather than performing deeper analysis,
the 4B model struggled with the complex prompt formats and defaulted entirely to outputting
a single class label.
DeBERTa-v3-base Training Disruption: DeBERTa-v3 failed entirely, having an F1 score of only
0.25. Despite isolating the text inputs, lowering the learning rate, and adding a 10% warm-up
phase, the model failed to converge and classified every single validation pair as Ambivalent.
This shows that both auto-regressive scaling (Qwen-4B) and modern encoder design (DeBERTa-
v3) run the risk of majority class regression when optimization stability or structural instruction
following breaks down.

## Positivity Bias and Length Anchoring
TF-IDF and Word2Vec models showed a clear vulnerability to superficial openings: when a
political speaker started the response with polite or positive phrases like "Yes" or "Thank you,"
the models categorized the instance as a Clear Reply, ignoring the signs of evasion.
The DistilBERT LengthAwareClassifier saw through this bias by anchoring text representations
to physical sequence metrics. By passing the normalized answer length directly to the classifi-
cation head, the model learned to flag long answers as indicative of the Ambivalent class.

# Synthesis & Strategic Recommendations
While splitting a task into subtasks reduces the load of an individual prompt, it increases the
number of failure points, as small models struggle to maintain context across JSON transitions.
To optimize performance on tasks like political evasion tracking:
Upgrade the Multi-Agent Foundation to the 2B: The Qwen3.5-2B model showed strong reasoning
when guided by Chain-of-Thought prompts (0.60 Macro F1). Rebuilding the D3-Agentic pipeline
using the 2B model would help prevent the context degradation and errors seen with the 0.8B
model.
Inject Structural Features into Agent Prompts: To replicate the success of the LengthAware-
Classifier, text metrics should be calculated and injected directly into the Gap and Evasion
Agent. This combines statistical feature strengths with generative reasoning.
