---
layout: page
title: "Names and Games"
description: The abundance of J-names in Super Smash Bros. Melee (alternatively, why J-Crew should invest in competitive Smash Brothers)
img:
importance: 4
category: fun
---

When you think of a first name, you might think of names like Chris, Sarah, Adam - but you invariably hit on a few such as John, Jessica, Justin, Jennifer (and the list goes on). This isn't entirely surprising - the most common starting letter of a first name in the United States is J. We can use some SQL queries to take a look at the actual numbers (sourced from BigQuery's public dataset of USA names). 

```sql
SELECT 
SUBSTR(name, 1, 1) as letter,
SUM(number) as total_names
FROM `bigquery-public-data.usa_names.usa_1910_current`
GROUP BY letter
ORDER BY total_names DESC
```
<div class="row justify-content-sm-center">
    <div class="col-sm-8 mt-3 mt-md-0">
        {% include figure.liquid path="assets/img/name_data.png" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
</div>

From the data between 1910 to 2021, we find that J names comprise about 13.25% of total names, or a little over 1 out of every 8 people. Cool! What are we doing with this information? 

Cut to: Super Smash Bros. Melee. Many born before 2000-ers may remember this game as either a fond relic of their childhood or an anger-inducing affair where they might have lost to their sibling one too many times. Currently, it boasts a thriving competitive community that hosts large tournaments and establishes a large ecosystem of players, spectators, commentators, and developers. 


<div class="row justify-content-sm-center">
    <div class="col-sm-8 mt-3 mt-md-0">
        {% include figure.liquid path="assets/img/evo.jpg" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Main stage of EVO, one of the biggest tournaments for Melee.
</div>

It's been an often-observed factoid for the Super Smash Bros. Melee community that a disproportionate amount of the best players in the world have first names starting with J. For example, long-time fan favorite Joseph "Mang0" Marquez and the loved and hated Juan "Hungrybox" Debiedma are both part of the J-crew. Even this year, 5 out of the top 10 players all proudly sport capital Js. How does this trend extend to more of the player base? Very conveniently, we have a carefully maintained yearly ranking of the top 100 players that we can use. With a little bit of effort to scrape the actual names and organize the data, we can compile a list of tables of each ranking list, complete with name, age, and nationality.  

```py
from bs4 import BeautifulSoup as bs, SoupStrainer as ss
import urllib.request
import pandas as pd
import concurrent.futures
import re

def scrape_player(tag):
    tag = urllib.parse.quote(tag.replace(" ", "_"))
    try:
        player_page = urllib.request.urlopen(("https://liquipedia.net/smash/") + (tag))
    except urllib.error.HTTPError as e:
        print(f"HTTP Error {e.code}: {e.reason}")
        print(tag)
        return [tag, "N/A", "N/A", "N/A"] 
    soup = bs(player_page, "lxml", parse_only = ss("div"))
    name = soup.find("div", class_ = "infobox-cell-2 infobox-description", text = re.compile(r"Name:"))
    nation = soup.find("div", class_ = "infobox-cell-2 infobox-description", text = "Nationality:")
    age = soup.find("div", class_ = "infobox-cell-2 infobox-description", text = re.compile(r"Born:"))
    list = [tag]
    if name:
        list.append(name.find_next().text)
    else:
        list.append("N/A")
    if nation:
        list.append(nation.find_next("a").find_next("a").text)
    else:
        list.append("N/A")
    if age:
        list.append(age.find_next().text[-3:-1])
    else:
        list.append("N/A")
    return list

yeartables = []
indices = [0, 2, 4, 5, 7, 9, 11, 13, 15]
url = "https://liquipedia.net/smash/SSBMRank"
dfs = pd.read_html(url)
colnames = ("Tag", "Name", "Nationality", "Age")
dfs[13].iloc[29, 1] = "null"
dfs[15].iloc[77, 1] = "null"
for k in indices:
    playertable = dfs[k].iloc[:, 1]
    with concurrent.futures.ThreadPoolExecutor() as executor:
        results = list(executor.map(scrape_player, playertable))
        yeartables.append(pd.DataFrame(results, columns = colnames))

```

<sup>(This code took two minutes to run so I would really appreciate any tips on scraping efficiency!)</sup>

```raw
               Tag                  Name    Nationality  Age
0            Mango        Joseph Marquez  United States   32
1           Armada         Adam Lindgren         Sweden   30
2         Mew2King       Jason Zimmerman  United States   35
3             PPMD          Kevin Nanney  United States   33
4        Hungrybox  Juan Manuel Debiedma  United States   30
..             ...                   ...            ...  ...
95  Bizzarro_Flame            Jason Yoon  United States  N/A
96          Hyprid          Kevin Bandel  United States  N/A
97            Vist                   N/A  United States  N/A
98        PKMVodka                   N/A         Canada  N/A
99             Ken                   N/A            N/A  N/A

[100 rows x 4 columns]
```

Filtering through only those entries with a name, age, and nationality, we can calculate the frequency of J-names in every year. We also calculate the average age of a player in a given year and include the rate of J-names for the corresponding birth year for comparison. Assuming a binomial distribution, we can also find the significance of each year's value and whether it truly is a peculiar pattern or just coincidence. 

```raw
    Age  National Average  Melee Frequency  Significance
0  1990             19.02            21.05       0.34480
1  1991             18.84            21.95       0.24370
2  1991             18.84            22.78       0.17361
3  1992             18.01            25.32       0.04980
4  1993             17.93            21.95       0.17490
5  1994             17.96            27.16       0.01632
6  1994             17.96            24.68       0.04840
7  1995             17.84            22.78       0.11380
8  1996             17.79            25.32       0.04396
```
There aren't many years that really stand out as interesting outliers - however, there are four years of 2016, 2018, 2019, and 2023 that have quite small significance values (<0.05). It's not a big enough trend to attribute any sort of grand J-name conspiracy to professional Melee players, but those few years definitely do pique my interest. 

<div class="row justify-content-sm-center">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid path="assets/img/trend.png" title="example image" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    National Averages are calculated using the average age of the top players for a given year.
</div>

Let's do a quick sanity check with another unrelated competitive scene with similar structures - the NBA. Using ESPN's lists of top 100 players from the 2020, 2022, and 2023 season (the ones I could easily find), we get values of 15%, 20%, and 21% for J-names. A bit more than what we might expect, but nothing too far away from the expected value, and a slightly bigger sample size too (less significance). 

Now, does this mean anything? Is there any rhyme or reason for this result (not that it's a particularly improbable one)? As far as I can tell, there doesn't seem to be an explanation - one could explore statistics on name trends for different backgrounds and demographics, but it seems harder to find that data. Perhaps there's a hidden correlation between performance and name starter, or even name and choice of character? While there may not be a satisfying conclusion to the data, there's still an attractive business opportunity for J-Crew to sponsor some of these players and form a "team" of their own. After all, there's nothing more appealing than a "name brand". 
