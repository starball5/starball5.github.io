This post was first published on 2024-06-22. It is licensed under [CC-BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/deed.en).

# heuristic development process story \#1

I'm a member of a volunteer working group in the [Stack Overflow](https://stackoverflow.com/) / [Stack Exchange](https://stackexchange.com/) community that develops heuristics for recognizing AI-Generated Content ("AIGC" for short) to support the site moderators in enforcing [the bans on AIGC on the platform](https://meta.stackexchange.com/a/384923/997587). AIGC suffers from ["hallucinations"](https://en.wikipedia.org/wiki/Hallucination_(artificial_intelligence)) and creates a host of other problems that motivated these bans. The aim of developing a heuristic in this working group is to find some well-defined characteristic(s) common to AIGC on the platform / in general, which support a high true-positive and low false-positive rate of matching (ideally automatable).

In this post, I want to peel back the curtain a tiny bit and show what work went into one particular heuristic which I championed the development of. Since [the heuristics are kept private](https://meta.stackexchange.com/q/391990/997587), I will only be describing my development process and avoid describing the heuristic itself. Apologies that many interesting details have been reduced to vague references. I have permission from the group to share what you see here.

One common structure of a response from an LLM like ChatGPT is `<redacted>`. There are several integer-valued characteristics of these `<redacted>` which LLMs "learned to prefer".

### Some history

I began noticing this pattern (and several others) in AIGC posts on Stack Overflow almost immediately after the release of ChatGPT on 2022-11-30. [The ban on AIGC on Stack Overflow](https://meta.stackoverflow.com/q/421831/11107541) began on 2022-12-05. I continued building an internal ability to recognize it, and applying that ability to flag policy violations for handling. Around 2023-06, [some controversy arose with Stack Overflow Inc. effectively preventing moderators from handling AIGC](https://meta.stackexchange.com/q/389811/997587), which [eventually was resolved](https://meta.stackexchange.com/q/391847/997587) in 2023-08, with the establishment of the current [(interim) policy on moderating AIGC](https://meta.stackexchange.com/q/391990/997587) with its heuristics framework, formal heuristic approval process (involving some fairly high expectations in accuracy and testing), and the working group I've been talking about. I immediately began work on developing and formalizing this heuristic for approval into this framework. It was a _heck_ of a lot of work, which finally came to fruition in formal approval roughly six months later on 2024-02-05- something I consider one of my proudest achievements so far (however silly that may sound).

### Research questions

In developing a heuristic to match against AIGC with this structure, I had two important questions:

- What is the [distribution](https://en.wikipedia.org/wiki/Probability_distribution) of these integer-valued characteristics across a large sample of instances of this pattern?
- Based on those distributions, given a piece of content and values it exhibits for those characteristics, what thresholds of confidence result in a _practical_ balance between not missing true-positives and avoiding false-positives?

### Deciding on a data source

The first real question was where to get a large sample of such posts. I had already been collecting a list of such actual pieces of content out in the wild from monitoring new Stack Overflow posts which matched my "internalized heuristic". In that set, I had roughly 700 entries. The other options would be to try to generate a corpus of such responses myself, or filter them out of an existing corpus. The latter seemed ideal for a "lab study" approach. The data would be clean of any of the messiness that happens when a human decides to make formatting and content adjustments before posting AIGC on Stack Overflow, and which happens when the output of an LLM goes through a Markdown sanitizer and rendering engine. But since not all LLM outputs match this heuristic, I'd still either have to go through the corpus myself to apply my "internalized heuristic". Automated filtering of an existing corpus would be pulling myself up from my bootstraps (the whole point was for sample data to result in parameter values for automated filtering- not the other way around). Add to that that I _wanted_ to be exposed to all the messiness of "real" pieces of content- manual edits, markup sanitization, and markup conversion/rendering and all, since the point of the heuristic is to deploy into the messiness of the real world. So I went with the list of 700 pieces of content I'd collected from the wild.

### Extracting the sample data

The tricky part about using that "real world" data from posts on Stack Overflow is that many of those posts had already been deleted for violating site policy (no thanks to myself), and I had only been pre-collecting their IDs and not their actual contents. For those which were not deleted yet, I could retrieve their Markdown contents using [the Stack Exchange Data Explorer](https://data.stackexchange.com/) (a public Microsoft SQL Server interface to public Stack Overflow data). For the rest, luckily, I'd already unlocked a high enough [privilege level](https://stackoverflow.com/help/privileges) on Stack Overflow to [be able to access the contents of deleted posts](https://stackoverflow.com/help/privileges/moderator-tools) given their ID (unfortunately, not to search across all of them), so I wrote a script to run in browser devtools using the Fetch API to essentially slowly scrape their original Markdown contents.

### Transforming/Cleaning the sample data

With the messiness of the real world (manual edits of "pure" LLM output, quirks from Markdown semantics, sanitization, and conversion to HTML), if I wanted to automate measurement of those integer-valued characteristics, I'd either have to adjust for every possible mutation or quirk (which would be impossible), or suck it up and clean up particularly messy sample data. Which I did. It was a chore, but it yielded a good intuition for how a basic implementation of automated detection could struggle with particular edge cases- areas which could use future work.

Posts on Stack Overflow are [written in CommonMark](https://meta.stackexchange.com/q/348746/997587) (plus some platform-specific extensions) and converted/rendered to HTML via markdig on the server side, and [markdown-it](https://www.npmjs.com/package/markdown-it) on the client side. Having extracted the original Markdown of the sample posts, I decided to focus on the rendered-HTML versions of the content, since it would help later with building tooling for annotation, and with implementing pattern-matching. To convert to HTML, I opted for markdown-it since I'm much more comfortable with JS than C# (though I've worked with both in professional contexts).

Another tricky decision point was how to deal with posts with multiple revisions. Posts on Stack Overflow can be edited, and copies of each revision are saved. Some of the posts in my dataset were originally AI-free with AIGC edited in later. Some were multiple revisions of AIGC, or edits with mixtures of "real" and "AI". This all came down to manual review and judgement calls.

I stored this pass of cleaned data in a single JSON file. With my SQL experience, I could have gone for a SQL database route, but for the types of calculations I'd be performing, and the relatively small size of data that I had, and the experimental/prototyping nature of what I was doing, it didn't make sense to me to get into that.

### Transforming/Annotating the sample data

The characteristic of LLM outputs this heuristic is designed to match has several variations which I intended to try to account for, and in general presented some challenges for automated measuring with certain ambiguous structures. Properly handling those ambiguities to avoid mis-measuring meant manually annotating the data. And again- the option to totally automate anything here was a no-go: The whole point of this is/was to inform proper automation.

Given my experience with web development and that I had the rendered HTML for each post, it made sense to build custom tooling (with a web-based UI) to make this annotation easier. Instead of total automation, I automated good guesses of proper annotation, and then manually intervened to confirm/correct where necessary. If I hadn't done that, this step would have taken on order of magnitude longer. Then I just converted the annotated data back from HTML to JSON and merged it with the pre-annotation data.

### Measuring/Calculating the characteristic values

With the data annotated, calculating the characteristic values was a breeze. I scripted this out in NodeJS. I got mean/expected values and standard deviations and other aggregate values, but this was relatively less enlightening than the distribution visualizations I made after this.

### Visualizing the characteristic-value distributions

I visualized the distributions of the characteristic values over the sample data points with [Kernel Density Estimation](https://en.wikipedia.org/wiki/Kernel_density_estimation). I used a standard normal density function, picking bandwidth values that I thought were appropriate for each one. I actually took a sort of janky route and used Desmos to plot the distribution function (got too lazy to code it up). It actually worked quite well.

It was fun looking at the data and seeing different peaks, plains, and valleys, which corresponded to different variations of the LLM output characteristic I was trying to match, and potentially with different LLMs or versions of related LLMs. Kind of a bummer I can't share the plots with you.

### Choosing confidence thresholds

My original ideal was to get more computationally friendly [approximations](https://en.wikipedia.org/wiki/Function_approximation) of my kernel density estimation, and then to base a confidence level that a piece of content matches the heuristic based on a naive product of plugging the measured/calculated values of the characteristics into their corresponding distribution functions (the characteristics are actually correlated, so this naive product wouldn't have been very correct to do, but hey- [I wouldn't be the first to do it anyway](https://en.wikipedia.org/wiki/Naive_Bayes_classifier)).

What I ended up doing was to just essentially mess around with a bunch of different min-max thresholds for each characteristic independently, and see how much of the data feel within those thresholds, and then go out into the wild and test them out to get a feel for how often false positives turned up (running search on posts created before the release of ChatGPT was a nice way to get a quick feel for this), and how many new true-positives were found. I used a combination of a first pass with [the Stack Exchange Data Explorer](https://data.stackexchange.com/) followed by a second pass with a NodeJS script to do more complicated calculation and filtering. A significant number of new true-positives were found- enough for me to quickly double the size of my collection. Given the number of "real" and AIGC posts I've seen in my curating activities on the Stack Overflow platform, it's usually easy to tell when something is a false positive. With time (and obviously the hard work of my peers and I), heuristics and counter-heuristics will continue to improve at filtering false-positives out. And I don't think it's ever been our or the moderators' or Stack Overflow Inc.'s intention to _fully_ automate handling of AIGC with no human oversight.

One of the limitations I had in this testing using the Stack Exchange Data Explorer was that I couldn't search through the contents of deleted posts (and had no other viable search tool that would let me do that at my disposal). So the search results I was looking at weren't totally representative in size or ratio of TP/FP. Only Stack Overflow Inc. staff have those kinds of tools at their disposal.

### Rinse and repeat

Our resident Stack Overflow Inc. staff member was quite busy with other things for a while, so a fair bit of time passed without much motion- except for me now being equipped with a pretty accurate new tool / search query for finding instances of LLM outputs matching this heuristic posted on Stack Overflow.

In that time, having doubled my true-positives found on Stack Overflow by applying the thresholds I'd found and from continuing to monitor new posts, I had a bit of a dilemma: Knowing that since LLMs continue to change, and that the distributions I found will likely shift over time, when I recalculate them with new data added, should I include the true positives found from applying the thresholds I was experimenting with? Or just those I found by other means from observing more neutrally in the wild? If I included targetted search results based on specific thresholds, I'd essentially be selectively amplifying the weight within those thresholds. Ironically, a similar problem/concern exists with the training of LLMs themselves: As AIGC fills the web, the next generations of LLMs will have to consider the effects of training on the outputs of prior generations, or find ways to identify that content and ignore it. I ended up deciding that since I'd continue to be on the lookout for matches outside of targetted searches, that would be sufficient to mitigate a really nasty amplification/loop effect. The distributions hadn't changed much.

### Final testing

At this point in the heuristic approval process, I started working more closely with a (now available) staff member from Stack Overflow Inc. to test the accuracy of my proposed heuristic on the full dataset including deleted posts. This involved porting my Stack Exchange Data Explorer query + NodeJS script combo to a [Snowflake](https://www.snowflake.com/en/) query (still using a combination of SQL and JS- just with a different database technology and its interface).

### Approval and deployment

From testing and a bit of final experimentation, the proposal was found to pass the baseline of our accuracy metric, and was approved. To my surprise, the heuristic was approved with a note that a post need not precisely fall within the exact thresholds we had settled on based on accuracy testing, and that if it matched the spirit of the proposal, it would count as an approved match, and that mods could trust their judgement. This was a bit surreal. That after all that work in seeking concreteness, precision, and quantification, what got approved was more along the lines of "we trust the mods to use their judgement on this. don't stress about the complicated details". Don't get me wrong- I was very happy to hear that. It was what I would have loved to hear all along. But I guess that's what tugged at me: "what was the point of all that work then?"

### Reflections

Of course, I understood and was aligned on the goal of the working group to be precise and to have a certain degree of discipline in defining and testing things. I haven't gone into the full details of the approval process or stringency requirements on testing, but I hope you can see that we really care about being responsible with defining and approving heuristics to avoid making wrong judgements / coming to wrong verdicts that something is AI-Generated. This was us "doing our homework" so to speak. We start out with observation and some qualitative intuition, make a hypothesis, test it, and verify it.

As for myself, through mulling on the question of "what was the point of all that work?", I came to appreciate a couple of things: I learned new marketable skills in the process. My existing skills were tested- building and using software to solve problems, processing data, communication, etc. I got to apply some stuff I learned in university that I thought I might never use. I got to work with some cool people. I led a project from ideation to deployment. It solidified what was already a growing realization that I enjoy working with data. I created a useful tool that will go on to have positive effects that potentially extend far beyond my little world.

So the process was well worth it. And however small the result might be in the grander scheme of things, I'm proud of it.

The work is not done here. This was just one little piece in a bigger picture of work that is largely hidden from public view. Hopefully by telling this story I was able to create a greater appreciation for what my peers and I are doing behind the scenes.
