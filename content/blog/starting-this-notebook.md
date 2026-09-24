\section*{Why Judge Models Are Interesting}

I have been looking at Jev, TypeSafe AI's ``System One'' model, and I think the interesting part is less the model itself and more the kind of problem it is designed to solve.

Most frontier models are increasingly good at reasoning. You give them a complicated problem, let them think through it, and get back an explanation.

But a lot of software does not need an explanation.

It needs a decision.

Imagine you run a company with a few thousand robots deployed in warehouses. Every day you receive hundreds of support reports:

\begin{quote}
Robot stopped halfway through a route.

Camera occasionally loses detections near the loading dock.

Battery only lasts three hours.

Robot hit a pallet after someone moved it into the aisle.
\end{quote}

You do not necessarily need a frontier reasoning model to write an analysis of every ticket.

The first thing you need is much simpler:

$$
\begin{aligned}
&\text{Is this urgent?}\\
&\text{Is this probably hardware, software, or environment?}\\
&\text{Should engineering look at it?}\\
&\text{How confident are we?}
\end{aligned}
$$

This is where judge models become interesting.

Jev is designed around structured judgments rather than arbitrary text generation. Instead of asking

$$
\text{``Analyze this support ticket.''}
$$

and then trying to extract a decision from three paragraphs of generated text, you can ask several bounded questions and receive structured answers with associated probabilities or confidence.

Conceptually, the pipeline becomes

$$
\text{ticket}
\rightarrow
\text{judgments}
\rightarrow
\text{routing decision}.
$$

For example:

$$
P(\text{urgent})=0.96,
$$

$$
\text{category}=\text{perception},
$$

$$
P(\text{requires engineering})=0.88.
$$

Now the software can actually do something with the result.

High-confidence perception failures might automatically enter the vision team's queue. Low-severity battery questions might go to support. Anything with low confidence can be sent to a person.

That last part is probably the most useful.

The goal does not need to be

$$
\text{AI makes every decision}.
$$

A much better system might be

$$
\begin{aligned}
\text{high confidence} &\rightarrow \text{automatic routing},\\
\text{low confidence} &\rightarrow \text{human review}.
\end{aligned}
$$

This is very different from trying to build one enormous agent that handles the entire support operation.

\subsection*{Why Not Just Use a Big LLM?}

You can.

But imagine doing this for 50,000 events every day.

A large reasoning model is useful when a ticket actually requires reasoning: comparing logs, understanding a strange failure, synthesizing several pieces of evidence, or proposing a fix.

It is probably overkill for

\begin{quote}
Does this look like a battery issue?
\end{quote}

or

\begin{quote}
Is this report urgent enough to wake someone up?
\end{quote}

There are enormous numbers of these small fuzzy decisions inside real software systems:

$$
\text{classify}
\rightarrow
\text{filter}
\rightarrow
\text{route}
\rightarrow
\text{escalate}.
$$

Today we often solve them either with brittle rules or by throwing a general-purpose LLM at the entire problem.

Judge models create an interesting middle ground.

They behave more like learned decision functions:

$$
f(x)
\rightarrow
(\text{decision},\text{confidence}).
$$

That is a very useful primitive.

\subsection*{The Important Catch}

Structured output does not mean the model cannot be wrong.

If the available answers are

$$
\{
\text{hardware},
\text{software},
\text{environment}
\},
$$

a judge can reliably return one of those three values and still choose the wrong one.

So

$$
\boxed{
\text{valid output}
\neq
\text{correct judgment}.
}
$$

This is why the evaluation dataset matters so much.

Before I let this fictional support system route production incidents, I would build a few hundred examples labeled by engineers:

$$
\text{ticket}
+
\text{correct category}
+
\text{correct severity}.
$$

Then I would measure the judge against those labels.

More importantly, I would measure accuracy as a function of confidence.

If tickets above \(95\%\) confidence are almost always correct, automate those.

If the model becomes unreliable below \(70\%\), send those to a person.

At that point confidence becomes useful because it controls how much autonomy the system gets.

\subsection*{The Bigger Idea}

What I find interesting about judge models is that they suggest a different way to build AI systems.

Not

$$
\boxed{\text{one huge model does everything}},
$$

but something closer to

$$
\text{code}
+
\text{fast judgments}
+
\text{reasoning models}
+
\text{humans}.
$$

Use normal software when the rule is deterministic.

Use a judge when the decision is fuzzy but narrow.

Use a larger model when the problem actually requires reasoning.

Use a person when uncertainty or consequence is high.

That feels much closer to how I expect practical agent systems to be built.

The interesting question may not be which model is smartest.

It may be figuring out exactly how much intelligence each decision actually needs.
