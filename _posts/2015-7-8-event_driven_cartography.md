---
title: Evis - event driven cartography with the GDELT dataset
author: Chester Holtz
layout: post
permalink: /blog/post/event_driven_cartography
categories:
  - Uncategorized
---

The progress on this project has been relatively slow, since i have been working on this project inconsistently. My interest in analysis of large data sets is due to my involvement in Professor Jiebo Luo's [VIStA (Visual Intelligence &amp; Social Multimedia Analytics) Research Group][1] as a research assistant. Currently, I am developing a web interface for the [gdelt (Global Database of Events, Language, and Tone) dataset][2] while also preforming analysis and rendering a visualization of real-time chaotic events - natural disasters, violence against civilians, etc. - in the world. 

Gdelt is a continuously updated repository which gathers events from around the world since 1979 and indexes them according to various metrics. Every 15 minutes, GDELT scrapes various news sources for relavent data and current events to update its database with entries containing locations, times, descriptions and other relavent data.

In testing, I use Python's matplotlib module with the basemap toolkit to plot data in real time and to literally draw correlations between events. Recently, GDELT was uploaded to google's bigquery which allows me to parse and analize over a quarter million events for free in seconds.


For the final rendition, I plan writting the application in javascript to be renderable in a browser.

The code is hosted on my [github][3], but is probably a little behind the current version as I am working on this project in a cloud vm.

An example demonstrating the power of google bigquery is provided below (taken from google's cloud platform blog): when I run the following sql query against a databse containing a quarter billion rows a set of 37 entries representing the most significant events of the last 37 years is returned in only a few seconds. From there, I can download the data as a json file and parse it, drawing data onto a map to be interacted with, or simply examined.
<pre class="prettyprint linenums">
SELECT Year, Actor1Name, Actor2Name, Count FROM (
SELECT Actor1Name, Actor2Name, Year, COUNT(*) Count, RANK() OVER(PARTITION BY YEAR ORDER BY Count DESC) rank
FROM 
(SELECT Actor1Name, Actor2Name,  Year FROM [gdelt-bq:full.events] WHERE Actor1Name < Actor2Name and Actor1CountryCode != '' and Actor2CountryCode != '' and Actor1CountryCode!=Actor2CountryCode),  (SELECT Actor2Name Actor1Name, Actor1Name Actor2Name, Year FROM [gdelt-bq:full.events] WHERE Actor1Name > Actor2Name  and Actor1CountryCode != '' and Actor2CountryCode != '' and Actor1CountryCode!=Actor2CountryCode),
WHERE Actor1Name IS NOT null
AND Actor2Name IS NOT null
GROUP EACH BY 1, 2, 3
HAVING Count > 100
)
WHERE rank=1
ORDER BY Year
</pre>

<pre class="prettyprint linenums">
{"Year":"1979","Actor1Name":"CHINA","Actor2Name":"VIETNAM","Count":"2668"}
{"Year":"1980","Actor1Name":"AFGHANISTAN","Actor2Name":"RUSSIA","Count":"3899"}
{"Year":"1981","Actor1Name":"RUSSIA","Actor2Name":"UNITED STATES","Count":"3079"}
{"Year":"1982","Actor1Name":"ISRAEL","Actor2Name":"LEBANON","Count":"4253"}
{"Year":"1983","Actor1Name":"ISRAEL","Actor2Name":"LEBANON","Count":"4955"}
...
{"Year":"2012","Actor1Name":"CHINA","Actor2Name":"UNITED STATES","Count":"42231"}
{"Year":"2013","Actor1Name":"RUSSIA","Actor2Name":"UNITED STATES","Count":"61191"}
{"Year":"2014","Actor1Name":"RUSSIA","Actor2Name":"UKRAINE","Count":"120995"}
{"Year":"2015","Actor1Name":"RUSSIA","Actor2Name":"UKRAINE","Count":"39236"}
</pre>

![alt text](/images/map.png "Example map")


The biggest issue encountered is the free data limit. Since the goal for this project involves a real-time component, it is necessary to requery data from gdelt every update.  

[1]: http://www.cs.rochester.edu/u/jluo/
[2]: http://gdeltproject.org/
[3]: https://github.com/Choltz95/Evis