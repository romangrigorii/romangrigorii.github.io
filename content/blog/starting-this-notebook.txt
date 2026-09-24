\section*{Why Judge Models Might Be More Useful Than Bigger Models}

I spent some time last night working with Jev, TypeSafe AI's new ``System One'' model, and I think the interesting part is not really Jev itself. It is the idea behind it.

Most of the current LLM race is focused on making models better at reasoning. Give the model a difficult problem, let it think for longer, and hopefully get a better answer.

That makes sense when the problem actually requires reasoning.

But a surprisingly large amount of software does not need another model to write an essay. It needs a model to make a decision.

Is this spam?

Is this lead actually interesting?

Should this agent call tool A or tool B?

Does this result need human review?

Which of these five categories does this belong to?

These are fundamentally different problems from asking a model to explain quantum mechanics or write a piece of software.

TypeSafe describes Jev as its first \emph{System One Model}, borrowing the terminology from Kahneman's distinction between fast, intuitive judgment and slower deliberative reasoning. Instead of generating arbitrary text, Jev accepts some state, answers bounded questions, and returns values that software can immediately act on.

The interface is essentially built around three kinds of questions:

$$
\begin{aligned}
\text{Noul} &:\quad \text{How likely is this statement to be true?}\\
\text{Choice} &:\quad \text{Which option should I select?}\\
\text{Score} &:\quad \text{Where does this fall on an ordered scale?}
\end{aligned}
$$

The API therefore behaves less like

$$
\text{prompt}
\rightarrow
\text{paragraph}
\rightarrow
\text{parse paragraph}
\rightarrow
\text{decision}
$$

and more like

$$
\text{state}
\rightarrow
\text{judgment}
\rightarrow
(\text{decision},\text{probabilities},\text{confidence}).
$$

That seems like a small distinction, but I think it matters a lot.

\subsection*{The Interesting Part Is the Confidence}

The structured output is useful, but the part I care about most is uncertainty.

If a classifier tells me

$$
P(\text{pain})=0.97,
$$

I might be comfortable allowing software to act automatically.

If it gives me

$$
P(\text{pain})=0.52,
$$

I probably want another model or a person to take a look.

That gives you a much more practical architecture:

$$
\begin{gathered}
\text{judgment}\\
\downarrow\\
\text{high confidence}
\rightarrow
\text{automatic action}\\
\\
\text{low confidence}
\rightarrow
\text{larger model / human review}.
\end{gathered}
$$

TypeSafe specifically trains Jev around calibrated decisions and returns probabilities and confidence with its structured outputs. Their intended workflow is exactly this: let software act above a threshold and escalate uncertain cases.

There is an important caveat here.

A confidence score is only useful if it is actually calibrated on something resembling your problem.

If predictions made with \(90\%\) confidence are only correct \(60\%\) of the time on your data, the number is not helping you very much.

So I would never take the confidence number at face value. I would measure it against my own evaluation set and determine where the useful operating thresholds actually are.

\subsection*{The Problem I Wanted to Solve}

The use case I was interested in was prospect qualification.

One thing I have learned is that a company simply ``doing AI'' is not particularly useful information.

Shipping an AI product is not pain.

Hiring ML engineers is not pain.

Putting ``AI-powered'' on the homepage is definitely not pain.

What I actually want to find is evidence that something is going wrong.

Maybe the company shipped an AI feature and users are complaining about it.

Maybe an agent is producing bad outputs publicly.

Maybe a feature had to be rolled back.

Maybe the company is hiring heavily around evaluation, reliability, or observability immediately after shipping a new system.

Those are much stronger signals.

The important distinction is

$$
\boxed{
\text{AI activity}
\neq
\text{AI pain}.
}
$$

The problem is that finding those signals manually takes a lot of time.

You might start with hundreds or thousands of companies, search through product launches, support threads, reviews, engineering posts, incidents, and customer complaints, and eventually find the handful that actually have a problem you might be able to solve.

That sounded like exactly the kind of bounded judgment problem that should not require a frontier reasoning model for every decision.

So I built the pipeline roughly as

$$
\text{company}
\rightarrow
\text{public evidence}
\rightarrow
\text{structured judgments}
\rightarrow
\text{confidence}
\rightarrow
\text{ranking}.
$$

Instead of asking one enormous prompt,

\begin{quote}
Is this company a good prospect for us and why?
\end{quote}

I decomposed the problem into smaller questions.

Is there evidence of a real AI product?

Is there evidence that it is currently causing problems?

How severe does the problem appear to be?

Is the evidence recent?

Does the problem actually match something we solve?

That decomposition turns one vague LLM task into several much narrower judgments.

This is also close to how TypeSafe describes its workflow approach: break a business process into narrow model judgments and deterministic code rather than asking one model to reason through the entire process in a single prompt. Their published workflow evaluations are explicitly constructed this way.

\subsection*{Version One Was Terrible}

The first version agreed with my own labels about \(10\%\) of the time.

Two out of twenty.

Obviously useless.

But that failure was probably the most valuable part of the exercise.

Once I looked at the disagreements, I found two problems.

The first was a hidden size filter. My pipeline was quietly rejecting companies that I personally considered strong targets.

The second problem was more interesting.

My ``pain detector'' was not detecting pain.

It was detecting AI news.

Those sound similar until you actually look at the output.

$$
\text{Company launches AI feature}
$$

is evidence of AI activity.

$$
\text{Company launches AI feature}
+
\text{users report failures}
+
\text{company rolls it back}
$$

is evidence of pain.

The classifier was doing a reasonable job answering the wrong question.

That reinforced something I think is extremely important when building agent systems:

$$
\boxed{
\text{A model can execute the wrong specification extremely well.}
}
$$

We have a tendency to blame the model whenever an AI system produces bad results.

Sometimes the model is the problem.

But sometimes the model is correctly optimizing exactly what we accidentally asked it to optimize.

\subsection*{Golden Labels Are What Made the Failure Visible}

The only reason I knew version one was bad was because I had already labeled examples myself.

For a small set of companies, I already knew what I considered a strong prospect, a weak prospect, and something that should not qualify at all.

Those became my golden labels.

Without them, the output actually looked reasonable.

That is the dangerous part.

An LLM can produce twenty classifications with twenty convincing explanations and make the whole system feel intelligent.

But

$$
\text{plausible output}
\neq
\text{correct system}.
$$

Once you have trusted labels, the development process becomes much more concrete:

$$
\text{prediction}
\rightarrow
\text{compare with label}
\rightarrow
\text{inspect disagreement}
\rightarrow
\text{change system}
\rightarrow
\text{evaluate again}.
$$

That is just ordinary machine learning evaluation, but I think agent development sometimes forgets it because the outputs look so convincing.

My bar should not be

\begin{quote}
These results look pretty good.
\end{quote}

It should be something measurable.

For example,

$$
\text{agreement with human labels},
$$

precision among the highest-ranked prospects, recall of known good prospects, and how accuracy changes as I increase the confidence threshold.

That last metric is especially important for a judge model.

If confidence is useful, I should be able to say something like:

$$
\text{only automate predictions where }c>0.9
$$

and see substantially better accuracy in that subset.

Otherwise the confidence number is just decoration.

\subsection*{``No Hallucinations'' Needs a Qualification}

One TypeSafe claim I would phrase carefully is that Jev has ``zero hallucinations.'' Their underlying point is legitimate but narrower than that phrase initially sounds. Jev's output space is defined in advance, so it cannot suddenly return a paragraph when the program expects an enum, invent a sixth option when there are five choices, or drift away from the requested schema. TypeSafe describes the absence of type errors as a property of the interface.

That does \emph{not} mean the judgment must be correct.

If the options are

$$
\{
\text{high pain},
\text{medium pain},
\text{low pain}
\},
$$

Jev will stay inside that set.

It can still confidently choose

$$
\text{high pain}
$$

when the correct answer is

$$
\text{low pain}.
$$

So I think the better distinction is

$$
\boxed{
\text{schema reliability}
\neq
\text{semantic correctness}.
}
$$

Both matter, but they are not the same thing.

\subsection*{Why Not Just Use Claude or GPT?}

You can absolutely use a general-purpose LLM for this.

In fact, structured-output modes have made doing so much easier.

The question is whether you actually need everything the larger model is capable of doing.

A reasoning model is extremely useful when I need to synthesize several pieces of evidence, plan something, write code, analyze an ambiguous situation, or generate a new answer.

But a production software system can contain thousands of much smaller branches:

$$
\text{classify},
\quad
\text{route},
\quad
\text{score},
\quad
\text{filter},
\quad
\text{verify},
\quad
\text{accept/reject}.
$$

Running a large reasoning model for every one of those branches can be a fairly expensive way of implementing a fuzzy \texttt{if} statement.

This is where Jev becomes interesting.

TypeSafe currently reports end-to-end latency of roughly \(70\)--\(500\) ms and an input price of $0.042 per million tokens. The company also reports much larger speed and cost advantages on its own workflow evaluations, although it explicitly notes that those results are workload-dependent and likely represent the higher end of expected gains.

Those numbers may change. The architecture is the more interesting part.

A judge becomes cheap enough that you can insert intelligence into places where you previously would have written a brittle rule.

Instead of

$$
\texttt{if keyword == "refund"}
$$

you can start asking something closer to

$$
P(
\text{customer is actually requesting a refund}
\mid
\text{conversation}
).
$$

That is a very powerful software primitive.

\subsection*{The Architecture I Find More Interesting}

I do not think the future is

$$
\boxed{\text{one enormous LLM does everything}.}
$$

I think a more practical system increasingly looks like

$$
\begin{gathered}
\text{deterministic code}
\\
+
\\
\text{fast learned judgments}
\\
+
\\
\text{reasoning models when reasoning is actually needed}
\\
+
\\
\text{humans when uncertainty or consequence is high}.
\end{gathered}
$$

The judge does not replace the reasoning model.

It decides when you need one.

That separation is what I find compelling.

You can imagine an agent loop where a cheap model is constantly answering questions such as

$$
\begin{aligned}
&\text{Did the previous tool call succeed?}\\
&\text{Is this result relevant?}\\
&\text{Do we have enough information to continue?}\\
&\text{Is this action risky?}\\
&\text{Does this need human review?}\\
&\text{Which tool should run next?}
\end{aligned}
$$

and only escalating to an expensive reasoning model when the problem actually deserves it.

At that point the judge is not really another chatbot.

It becomes part of the control system.

\subsection*{My Main Takeaway}

The thing I took away from this experiment is not that Jev is going to replace Claude, GPT, or other frontier models.

It is almost the opposite.

I think we have been using general-purpose LLMs for a lot of jobs that do not require general-purpose generation.

There is an important difference between

$$
\text{generate an answer}
$$

and

$$
\text{make a judgment}.
$$

Once those become separate primitives, you can build systems differently.

Use code for things code is good at.

Use a fast judge for fuzzy decisions.

Use large reasoning models when the problem actually requires reasoning.

Send uncertain or consequential cases to a person.

And evaluate the entire thing against labels you trust.

The more I work with these systems, the more I think the difficult part is becoming less about simply finding the ``smartest model'' and more about deciding exactly \emph{where intelligence belongs in the system}.

A good judge is useful.

A calibrated judge is much more useful.

But neither matters very much if you do not have a good definition of what the correct judgment was supposed to be in the first place.

$$
\boxed{
\text{better automation}
=
\text{better task definition}
+
\text{better judgments}
+
\text{better evaluation}.
}
$$
