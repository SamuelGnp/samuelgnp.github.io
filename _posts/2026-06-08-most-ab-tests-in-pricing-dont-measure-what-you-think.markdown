---
title: "Most A/B tests in pricing don't measure what you think they measure"
layout: post
date: 2026-06-08 10:00
image: /assets/images/markdown.jpg
headerImage: false
tag:
  - ab-testing
  - pricing
  - causal-inference
category: blog
author: samuelgnap
description: Unit-level randomization breaks in pricing because demand interferes across products, search results, and competitor reactions, so most reported lifts are biased upward.
---

One product family in a test showed a clean double-digit uplift. The team was ready to celebrate. Then someone pulled the global revenue chart for the same window and the line was flat. Not slightly flat. Flat in the way that tells you the win on one product was paid for, in full, by losses you were not measuring. We spent the rest of the week working out how to explain that to leadership without sounding like we had wasted a quarter.

That gap between the local treatment effect and the thing that pays the bills is, in my experience, the central problem with A/B testing in pricing. The math is fine. The framework is fine. The assumption underneath it is not.

## The assumption A/B testing rests on

The standard A/B test assumes the response of one unit does not depend on the treatment assigned to any other unit. That is the Stable Unit Treatment Value Assumption, SUTVA in shorthand. If I randomly assign customers to a new price and you to the old one, my purchase decision must have no effect on yours. The estimator only recovers the causal effect if the world holds still around each unit while you measure it.

Most experimentation playbooks treat SUTVA as a footnote. In digital advertising it usually is one, because users do not see each other's ads in any meaningful way and the system does not adjust between them. In pricing, SUTVA is the entire problem.

## Three ways pricing breaks it

The first way is cross-product substitution. Customers in pricing do not buy a single good in isolation. They choose between a bundle of related products. If I cut the price on product A, some share of the people who would have bought product B switch over. The treatment effect on A looks clean. The total wallet has not moved. The bigger the assortment, and the closer the substitutes, the worse this gets. In airlines it is the difference between testing a fare class and testing the route as a whole. The fare class result will always look better than the route result, because part of the lift is just demand walking next door.

The second way is search-result interference. Customers shop a list. They sort by price, by departure time, by total duration. The moment you change the price of one item, the ranking of everything around it shifts. The control unit on row four is now on row two. Its conversion rate changes for a reason that has nothing to do with its own price. The thing you wanted to hold constant is precisely the thing your treatment moved. Anyone who has worked on marketplace experiments has lived this. Anyone who has not tends to find out late.

The third way is competitor reaction. In a watched market, your prices are public information and the people watching are not customers. They are pricing teams at the other carriers. The instant your treatment goes live, prices around it adjust. Sometimes it takes hours. Sometimes it takes a day. Either way, the period in which the treatment is running is no longer ceteris paribus. You are not measuring the effect of your price change against the old world. You are measuring it against the new world your price change called into existence.

## What that means for reported lifts

If you accept those three mechanisms, you have to accept that the lifts coming out of standard pricing A/B tests are systematically biased upward. Cross-product substitution inflates them. Ranking effects inflate them in the direction the treatment pushes. Competitor reaction inflates them in the short window before the market re-equilibrates and then often deflates them later, after the test is already closed.

I do not have a single clean number for the size of that bias and neither does anyone else I have worked with. The honest read is that the bias is rarely zero and rarely small. When I read a pricing experiment report from inside or outside my own team, I now do a quiet mental haircut on the headline number before I think about anything else. The haircut is not a sophisticated correction. It is the price of admission for thinking clearly about results from a method that does not, strictly speaking, apply.

## What I use instead

None of the alternatives are clean. They each trade one set of assumptions for another, and the right choice depends on which violation hurts the most in your specific setting.

Switchback designs randomize within a unit over time rather than across units at a moment. The whole market sees the treatment for a window, then the control for a window, then the treatment again. This handles cross-product and ranking interference well, because the comparison is within the same shopping context. It does nothing for competitor reaction and it introduces temporal confounding you have to model out. The standard error story is also worse, because the effective sample size is closer to the number of switches than the number of bookings.

Geo splits assume one geographic region is isolated enough from another that the treatment in one does not bleed into the other. For some products this is defensible. For others it is wishful thinking dressed up in a map. Airlines are a hard case because customers shop across geos by default and competitors operate in both.

Synthetic control builds a counterfactual for one treated unit out of a weighted combination of untreated ones. It is the method I reach for when the data shape is one treated route, or one treated product, against a panel of plausible controls. It does not pretend to be an A/B test. It estimates a counterfactual path, and the pre-treatment fit is testable in a way that parallel-trends assumptions are not. The cost is that it does not give you a clean p-value and stakeholders do not always know what to do with the output.

Each method has its own pathology. None of them recovers the answer a perfect randomized experiment would have given you, because that experiment does not exist for this problem.

## The thing nobody says out loud

A version of this post that played it safe would stop at "be careful about interference". I think the implication is stronger than that. Most published lift numbers from pricing experiments, mine and other people's, are biased upward by a meaningful amount. The size of the bias is not known industry-wide, because correcting it requires running the experiment with and without interference, which almost no one does. The result is that the literature, the conference talks, and the CV bullet points all point in the same direction. Up.

I am not arguing that pricing teams are dishonest. I am arguing that the standard method does not measure what it claims to measure, and that the bias has a sign. If everyone is using the same tool with the same flaw, everyone's numbers tilt the same way.

## The hiring implication

When a candidate for a pricing role quotes a percentage lift on a CV, the number itself is a weak signal. The interesting signal is what comes after the follow-up question. How was the experiment designed? What did you do about cross-product substitution? Did you check for competitor reaction during the treatment window? Did the global metric move in line with the local one?

A candidate who has thought about this answers calmly and specifically. A candidate who has not gets defensive about the number. Both responses are informative and the second one is usually the more useful of the two.

## Closing read

An A/B test in pricing assumes the world holds still while you run it. The world never does. The methods that survive contact with that fact are messier than randomization and harder to communicate, and they are the methods I trust. Haircut every reported pricing lift in your head, including the ones with your name on them. The discipline of doing that out loud, in a meeting, in front of the people who want the number to be real, is one of the more useful habits a senior practitioner can build.
