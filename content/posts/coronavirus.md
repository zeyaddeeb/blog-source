---
date: 2020-02-22
title: "Coronavirus (COVID-19) outbreak analysis"
markup: mmark
---

The [first news article I read this morning](https://www.bloomberg.com/news/articles/2020-02-22/coronavirus-may-be-the-disease-x-health-agency-warned-about) described Coronavirus (COVID-19) as "disease X";

> The World Health Organization cautioned years ago that a mysterious “disease X” could spark an international contagion. The new coronavirus, with its ability to quickly morph from mild to deadly, is emerging as a contender - Bloomberg

This is some scary shit! I've been working on some models attempting to have a objective look at COVID-19 outbreak, which I'll be sharing today.

John Hopkins: The Center for Systems Science and Engineering (CSSE), is kind enough to share data they collect from multiple sources. You can grab the data from this [link](https://gisanddata.maps.arcgis.com/apps/opsdashboard/index.html#/bda7594740fd40299423467b48e9ecf6) or see my [Github](https://github.com/zeyaddeeb/coronavirus).

Obviously, the first thing is to look at what the trend;

{{% plot "/plots/coronavirus-trend.html" %}}

This seems like confirmed cases are on the rise with total of whopping 1,058,633 cases, while recovery rate is at 13.43% and the death rate 2.46% as of Feb 26 2020.

If you have lower than normal empathy, you might say that not too shabby, only 26k or so people died... So why is everyone freaking out?!

{{% plot "/plots/coronavirus-map.html" %}}



### SIR epidemic model

$$
S(t)+I(t)+R(t)=N
$$

where  $$t$$ is time.
