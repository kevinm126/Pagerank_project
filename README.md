# Pagerank Project

In this project, you will create a simple search engine for the website <https://www.lawfareblog.com>.
This website provides legal analysis on US national security issues.

**Due date:** Sunday, 18 September at midnight

**Computation:**
This project has low computational requirements.
You are not required to complete it on the lambda server (although you are welcome to if you'd like).

## Background

**Data:**

The `data` folder contains two files that store example "web graphs".
The file `small.csv.gz` contains the example graph from the *Deeper Inside Pagerank* paper.
This is a small graph, so we can manually inspect the contents of this file with the following command:
```
$ zcat data/small.csv.gz
source,target
1,2
1,3
3,1
3,2
3,5
4,5
4,6
5,6
5,4
6,4
```

> **Recall:**
> The `cat` terminal command outputs the contents of a file to stdout, and the `zcat` command first decompressed a gzipped file and then outputs the decompressed contents.

As you can see, the graph is stored as a CSV file.
The first line is a header,
and each subsequent line stores a single edge in the graph.
The first column contains the source node of the edge and the second column the target node.
The file is assumed to be sorted alphabetically.

The second data file `lawfareblog.csv.gz` contains the link structure for the lawfare blog.
Let's take a look at the first 10 of these lines:
```
$ zcat data/lawfareblog.csv.gz | head
source,target
www.lawfareblog.com/,www.lawfareblog.com/topic/interrogation
www.lawfareblog.com/,www.lawfareblog.com/upcoming-events
www.lawfareblog.com/,www.lawfareblog.com/
www.lawfareblog.com/,www.lawfareblog.com/our-comments-policy
www.lawfareblog.com/,www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
www.lawfareblog.com/,www.lawfareblog.com/topic/lawfare-research-paper-series
www.lawfareblog.com/,www.lawfareblog.com/topic/book-reviews
www.lawfareblog.com/,www.lawfareblog.com/documents-related-mueller-investigation
www.lawfareblog.com/,www.lawfareblog.com/topic/international-law-loac
```
You can see that in this file, the node names are URLs.
Semantically, each line corresponds to an HTML `<a>` tag that is contained in the source webpage and links to the target webpage.

We can use the following command to count the total number of links in the file:
```
$ zcat data/lawfareblog.csv.gz | wc -l
1610789
```
Since every link corresponds to a non-zero entry in the `P` matrix,
this is also the value of `nnz(P)`.
(Technically, we should subtract 1 from this value since the `wc -l` command also counts the header line, not just the data lines.)

To get the dimensions of `P`, we need to count the total number of nodes in the graph.
The following command achieves this by: decompressing the file, extracting the first column, removing all duplicate lines, then counting the results.
```
$ zcat data/lawfareblog.csv.gz | cut -f1 -d, | uniq | wc -l
25761
```
This matrix is large enough that computing matrix products for dense matrices takes several minutes on a single CPU.
Fortunately, however, the matrix is very sparse.
The following python code computes the fraction of entries in the matrix with non-zero values:
```
>>> 1610788 / (25760**2)
0.0024274297384360172
```
Thus, by using sparse matrix operations, we will be able to speed up the code significantly.

**Code:**

The `pagerank.py` file contains code for loading the graph CSV files and searching through their nodes for key phrases.
For example, you can perform a search for all nodes (i.e. urls) that mention the string `corona` with the following command:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --search_query=corona
```

> **NOTE:**
> It will take about 10 seconds to load and parse the data files.
> All the other computation happens essentially instantly.

Currently, the pagerank of the nodes is not currently being calculated correctly, and so the webpages are returned in an arbitrary order.
Your task in this assignment will be to fix these calculations in order to have the most important results (i.e. highest pagerank results) returned first.

## Task 1: the power method

Implement the `WebGraph.power_method` function in `pagerank.py` for computing the pagerank vector by fixing the `FIXME` annotation.

**Part 1:**

To check that your implementation is working,
you should run the program on the `data/small.csv.gz` graph.
For my implementation, I get the following output.
```
$ python3 pagerank.py --data=data/small.csv.gz --verbose
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 residual=2.5629e-01
DEBUG:root:i=1 residual=1.1841e-01
DEBUG:root:i=2 residual=7.0701e-02
DEBUG:root:i=3 residual=3.1815e-02
DEBUG:root:i=4 residual=2.0497e-02
DEBUG:root:i=5 residual=1.0108e-02
DEBUG:root:i=6 residual=6.3716e-03
DEBUG:root:i=7 residual=3.4228e-03
DEBUG:root:i=8 residual=2.0879e-03
DEBUG:root:i=9 residual=1.1750e-03
DEBUG:root:i=10 residual=7.0131e-04
DEBUG:root:i=11 residual=4.0321e-04
DEBUG:root:i=12 residual=2.3800e-04
DEBUG:root:i=13 residual=1.3812e-04
DEBUG:root:i=14 residual=8.1083e-05
DEBUG:root:i=15 residual=4.7251e-05
DEBUG:root:i=16 residual=2.7704e-05
DEBUG:root:i=17 residual=1.6164e-05
DEBUG:root:i=18 residual=9.4778e-06
DEBUG:root:i=19 residual=5.5066e-06
DEBUG:root:i=20 residual=3.2042e-06
DEBUG:root:i=21 residual=1.8612e-06
DEBUG:root:i=22 residual=1.1283e-06
DEBUG:root:i=23 residual=6.1907e-07
INFO:root:rank=0 pagerank=6.6270e-01 url=4
INFO:root:rank=1 pagerank=5.2179e-01 url=6
INFO:root:rank=2 pagerank=4.1434e-01 url=5
INFO:root:rank=3 pagerank=2.3175e-01 url=2
INFO:root:rank=4 pagerank=1.8590e-01 url=3
INFO:root:rank=5 pagerank=1.6917e-01 url=1
```
Yours likely won't be identical (due to weird floating point issues), but it should be similar.
In particular, the ranking of the nodes/urls should be the same order.

> **NOTE:**
> The `--verbose` flag causes all of the lines beginning with `DEBUG` to be printed.
> By default, only lines beginning with `INFO` are printed.

**Part 2:**

The `pagerank.py` file has an option `--search_query`, which takes a string as a parameter.
If this argument is used, then the program returns all nodes that match the query string sorted according to their pagerank.
Essentially, this gives us the most important pages related to our query.

Again, you may not get the exact same results as me,
but you should get similar results to the examples I've shown below.
Verify that you do in fact get similar results.

```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='corona'
INFO:root:rank=0 pagerank=1.0038e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=1 pagerank=8.9224e-04 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=2 pagerank=7.0390e-04 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=3 pagerank=6.9153e-04 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=4 pagerank=6.7041e-04 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=5 pagerank=6.6256e-04 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
INFO:root:rank=6 pagerank=6.5046e-04 url=www.lawfareblog.com/congressional-homeland-security-committees-seek-ways-support-state-federal-responses-coronavirus
INFO:root:rank=7 pagerank=6.3620e-04 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
INFO:root:rank=8 pagerank=6.1248e-04 url=www.lawfareblog.com/house-subcommittee-voices-concerns-over-us-management-coronavirus
INFO:root:rank=9 pagerank=6.0187e-04 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='trump'
INFO:root:rank=0 pagerank=5.7826e-03 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=5.2338e-03 url=www.lawfareblog.com/document-trump-revokes-obama-executive-order-counterterrorism-strike-casualty-reporting
INFO:root:rank=2 pagerank=5.1297e-03 url=www.lawfareblog.com/trump-administrations-worrying-new-policy-israeli-settlements
INFO:root:rank=3 pagerank=4.6599e-03 url=www.lawfareblog.com/dc-circuit-overrules-district-courts-due-process-ruling-qasim-v-trump
INFO:root:rank=4 pagerank=4.5934e-03 url=www.lawfareblog.com/donald-trump-and-politically-weaponized-executive-branch
INFO:root:rank=5 pagerank=4.3071e-03 url=www.lawfareblog.com/how-trumps-approach-middle-east-ignores-past-future-and-human-condition
INFO:root:rank=6 pagerank=4.0935e-03 url=www.lawfareblog.com/why-trump-cant-buy-greenland
INFO:root:rank=7 pagerank=3.7591e-03 url=www.lawfareblog.com/oral-argument-summary-qassim-v-trump
INFO:root:rank=8 pagerank=3.4509e-03 url=www.lawfareblog.com/dc-circuit-court-denies-trump-rehearing-mazars-case
INFO:root:rank=9 pagerank=3.4484e-03 url=www.lawfareblog.com/second-circuit-rules-mazars-must-hand-over-trump-tax-returns-new-york-prosecutors

$ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='iran'
INFO:root:rank=0 pagerank=4.5746e-03 url=www.lawfareblog.com/praise-presidents-iran-tweets
INFO:root:rank=1 pagerank=4.4174e-03 url=www.lawfareblog.com/how-us-iran-tensions-could-disrupt-iraqs-fragile-peace
INFO:root:rank=2 pagerank=2.6928e-03 url=www.lawfareblog.com/cyber-command-operational-update-clarifying-june-2019-iran-operation
INFO:root:rank=3 pagerank=1.9391e-03 url=www.lawfareblog.com/aborted-iran-strike-fine-line-between-necessity-and-revenge
INFO:root:rank=4 pagerank=1.5452e-03 url=www.lawfareblog.com/parsing-state-departments-letter-use-force-against-iran
INFO:root:rank=5 pagerank=1.5357e-03 url=www.lawfareblog.com/iranian-hostage-crisis-and-its-effect-american-politics
INFO:root:rank=6 pagerank=1.5258e-03 url=www.lawfareblog.com/announcing-united-states-and-use-force-against-iran-new-lawfare-e-book
INFO:root:rank=7 pagerank=1.4221e-03 url=www.lawfareblog.com/us-names-iranian-revolutionary-guard-terrorist-organization-and-sanctions-international-criminal
INFO:root:rank=8 pagerank=1.1788e-03 url=www.lawfareblog.com/iran-shoots-down-us-drone-domestic-and-international-legal-implications
INFO:root:rank=9 pagerank=1.1463e-03 url=www.lawfareblog.com/israel-iran-syria-clash-and-law-use-force
```

**Part 3:**

The webgraph of lawfareblog.com (i.e. the `P` matrix) naturally contains a lot of structure.
For example, essentially all pages on the domain have links to the root page <https://lawfareblog.com/> and other "non-article" pages like <https://www.lawfareblog.com/topics> and <https://www.lawfareblog.com/subscribe-lawfare>.
These pages therefore have a large pagerank.
We can get a list of the pages with the largest pagerank by running

```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz
INFO:root:rank=0 pagerank=2.8741e-01 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=1 pagerank=2.8741e-01 url=www.lawfareblog.com/masthead
INFO:root:rank=2 pagerank=2.8741e-01 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
INFO:root:rank=3 pagerank=2.8741e-01 url=www.lawfareblog.com/documents-related-mueller-investigation
INFO:root:rank=4 pagerank=2.8741e-01 url=www.lawfareblog.com/topics
INFO:root:rank=5 pagerank=2.8741e-01 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=6 pagerank=2.8741e-01 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=7 pagerank=2.8741e-01 url=www.lawfareblog.com/support-lawfare
INFO:root:rank=8 pagerank=2.8741e-01 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 pagerank=2.8741e-01 url=www.lawfareblog.com/our-comments-policy
```

Most of these pages are not very interesting, however, because they are not articles,
and usually when we are performing a web search, we only want articles.

This raises the question: How can we find the most important articles filtering out the non-article pages?
The answer is to modify the `P` matrix by removing all links to non-article pages.

This raises another question: How do we know if a link is a non-article page?
Unfortunately, this is a hard question to answer with 100% accuracy,
but there are many methods that get us most of the way there.
One easy to implement method is to compute what's called the "in-link ratio" of each node (i.e. the total number of edges with the node as a target divided by the total number of nodes),
and then remove nodes from the search results with too-high of a ratio.
The intuition is that non-article pages often appear in the menu of a webpage, and so have links from almost all of the other webpages;
but article-webpages are unlikely to appear on a menu and so will only have a small number of links from other webpages.
The `--filter_ratio` parameter causes the code to remove all pages that have an in-link ratio larger than the provided value.

Using this option, we can estimate the most important articles on the domain with the following command:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2
INFO:root:rank=0 pagerank=3.4696e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=2.9521e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=5 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=6 pagerank=1.5071e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 pagerank=1.4957e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 pagerank=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 pagerank=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull
```
Notice that the urls in this list look much more like articles than the urls in the previous list.

When Google calculates their `P` matrix for the web,
they use a similar (but much more complicated) process to modify the `P` matrix in order to reduce spam results.
The exact formula they use is a jealously guarded secret that they update continuously.

In the case above, notice that we have accidentally removed the blog's most popular article (<www.lawfareblog.com/snowden-revelations>).
The blog editors believed that Snowden's revelations about NSA spying are so important that they directly put a link to the article on the menu.
So every single webpage in the domain links to the Snowden article,
and our "anti-spam" `--filter-ratio` argument removed this article from the list.
In general, it is a challenging open problem to remove spam from pagerank results,
and all current solutions rely on careful human tuning and still have lots of false positives and false negatives.

**Part 4:**

Recall from the reading that the runtime of pagerank depends heavily on the eigengap of the `\bar\bar P` matrix,
and that this eigengap is bounded by the alpha parameter.

Run the following four commands:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose 
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --alpha=0.99999
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
```
You should notice that the last command takes considerably more iterations to compute the pagerank vector.
(My code takes 685 iterations for this call, and about 10 iterations for all the others.)

This raises the question: Why does the second command (with the `--alpha` option but without the `--filter_ratio`) option not take a long time to run?
The answer is that the `P` graph for <https://www.lawfareblog.com> naturally has a large eigengap and so is fast to compute for all alpha values,
but the modified graph does not have a large eigengap and so requires a small alpha for fast convergence.

Changing the value of alpha also gives us very different pagerank rankings.
For example, 
```
$ python3 pagerank_solution.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
INFO:root:rank=0 pagerank=3.4696e-01 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=2.9521e-01 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=2.9040e-01 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=1.5179e-01 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=5 pagerank=1.5099e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=6 pagerank=1.5071e-01 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 pagerank=1.4957e-01 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 pagerank=1.4367e-01 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 pagerank=1.4240e-01 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull

$ python3 pagerank_solution.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
INFO:root:rank=0 pagerank=7.0149e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=7.0149e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.0552e-01 url=www.lawfareblog.com/cost-using-zero-days
INFO:root:rank=3 pagerank=3.1755e-02 url=www.lawfareblog.com/lawfare-podcast-former-congressman-brian-baird-and-daniel-schuman-how-congress-can-continue-function
INFO:root:rank=4 pagerank=2.2040e-02 url=www.lawfareblog.com/events
INFO:root:rank=5 pagerank=1.6027e-02 url=www.lawfareblog.com/water-wars-increased-us-focus-indo-pacific
INFO:root:rank=6 pagerank=1.6026e-02 url=www.lawfareblog.com/water-wars-drill-maybe-drill
INFO:root:rank=7 pagerank=1.6023e-02 url=www.lawfareblog.com/water-wars-disjointed-operations-south-china-sea
INFO:root:rank=8 pagerank=1.6020e-02 url=www.lawfareblog.com/water-wars-song-oil-and-fire
INFO:root:rank=9 pagerank=1.6020e-02 url=www.lawfareblog.com/water-wars-sinking-feeling-philippine-china-relations
```

Which of these rankings is better is entirely subjective,
and the only way to know if you have the "best" alpha for your application is to try several variations and see what is best.
If large alphas are good for your application, you can see that there is a trade-off between quality answers and algorithmic runtime.
We'll be exploring this trade-off more formally in class over the rest of the semester.

## Task 2: the personalization vector

The most interesting applications of pagerank involve the personalization vector.
Implement the `WebGraph.make_personalization_vector` function so that it outputs a personalization vector tuned for the input query.
The pseudocode for the function is:
```
for each index in the personalization vector:
    get the url for the index (see the _url_to_index function)
    check if the url satisfies the input query (see the url_satisfies_query function)
    if so, set the corresponding index to one
normalize the vector
```

**Part 1:**

The command line argument `--personalization_vector_query` will use the function you created above to augment your search with a custom personalization vector.
If you've implemented the function correctly,
you should get results similar to:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona'
INFO:root:rank=0 pagerank=6.3127e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=6.3124e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.5947e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 pagerank=1.2209e-01 url=www.lawfareblog.com/brexit-not-immune-coronavirus
INFO:root:rank=4 pagerank=1.2209e-01 url=www.lawfareblog.com/rational-security-my-corona-edition
INFO:root:rank=5 pagerank=9.3360e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=6 pagerank=9.1920e-02 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=7 pagerank=9.1920e-02 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=8 pagerank=7.7770e-02 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=9 pagerank=7.2888e-02 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
```

Notice that these results are significantly different than when using the `--search_query` option:
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --search_query='corona'
INFO:root:rank=0 pagerank=8.1320e-03 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=1 pagerank=7.7908e-03 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=2 pagerank=5.2262e-03 url=www.lawfareblog.com/livestream-house-oversight-committee-holds-hearing-government-coronavirus-response
INFO:root:rank=3 pagerank=3.9584e-03 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=4 pagerank=3.8114e-03 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=5 pagerank=3.3973e-03 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
INFO:root:rank=6 pagerank=3.3633e-03 url=www.lawfareblog.com/cyberlaw-podcast-how-israel-fighting-coronavirus
INFO:root:rank=7 pagerank=3.3557e-03 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=8 pagerank=3.2160e-03 url=www.lawfareblog.com/congress-needs-coronavirus-failsafe-its-too-late
INFO:root:rank=9 pagerank=3.1036e-03 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
```

Which results are better?
Again, that depends on what you mean by "better."
With the `--personalization_vector_query` option,
a webpage is important only if other coronavirus webpages also think it's important;
with the `--search_query` option,
a webpage is important if any other webpage thinks it's important.
You'll notice that in the later example, many of the webpages are about Congressional proceedings related to the coronavirus.
From a strictly coronavirus perspective, these are not very important webpages.
But in the broader context of national security, these are very important webpages.

Google engineers spend TONs of time fine-tuning their pagerank personalization vectors to remove spam webpages.
Exactly how they do this is another one of their secrets that they don't publicly talk about.

**Part 2:**

Another use of the `--personalization_vector_query` option is that we can find out what webpages are related to the coronavirus but don't directly mention the coronavirus.
This can be used to map out what types of topics are similar to the coronavirus.

For example, the following query ranks all webpages by their `corona` importance,
but removes webpages mentioning `corona` from the results.
```
$ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona' --search_query='-corona'
INFO:root:rank=0 pagerank=6.3127e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=6.3124e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.5947e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 pagerank=9.3360e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=4 pagerank=7.0277e-02 url=www.lawfareblog.com/fault-lines-foreign-policy-quarantined
INFO:root:rank=5 pagerank=6.9713e-02 url=www.lawfareblog.com/lawfare-podcast-mom-and-dad-talk-clinical-trials-pandemic
INFO:root:rank=6 pagerank=6.4944e-02 url=www.lawfareblog.com/limits-world-health-organization
INFO:root:rank=7 pagerank=5.9492e-02 url=www.lawfareblog.com/chinatalk-dispatches-shanghai-beijing-and-hong-kong
INFO:root:rank=8 pagerank=5.1245e-02 url=www.lawfareblog.com/us-moves-dismiss-case-against-company-linked-ira-troll-farm
INFO:root:rank=9 pagerank=5.1245e-02 url=www.lawfareblog.com/livestream-house-armed-services-holds-hearing-national-security-challenges-north-and-south-america
```
You can see that there are many urls about concepts that are obviously related like "covid", "clinical trials", and "quarantine",
but this algorithm also finds articles about Chinese propaganda and Trump's policy decisions.
Both of these articles are highly relevant to coronavirus discussions,
but a simple keyword search for corona or related terms would not find these articles.

<!--
**Part 3:**

Select another topic related to national security.
You should experiment with a national security topic other than the coronavirus.
For example, find out what articles are important to the `iran` topic but do not contain the word `iran`.
Your goal should be to discover what topics that www.lawfareblog.com considers to be related to the national security topic you choose.
-->

## Submission

1. Create a new repo on github (not a fork of this repo).

1. Run the following commands, and paste their output into the code blocks below.

   Task 1, part 1:
```   
$ python3 pagerank.py --data=data/small.csv.gz --verbose                   
DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 accuracy=tensor(0.2563)
DEBUG:root:i=1 accuracy=tensor(0.1184)
DEBUG:root:i=2 accuracy=tensor(0.0707)
DEBUG:root:i=3 accuracy=tensor(0.0318)
DEBUG:root:i=4 accuracy=tensor(0.0205)
DEBUG:root:i=5 accuracy=tensor(0.0101)
DEBUG:root:i=6 accuracy=tensor(0.0064)
DEBUG:root:i=7 accuracy=tensor(0.0034)
DEBUG:root:i=8 accuracy=tensor(0.0021)
DEBUG:root:i=9 accuracy=tensor(0.0012)
DEBUG:root:i=10 accuracy=tensor(0.0007)
DEBUG:root:i=11 accuracy=tensor(0.0004)
DEBUG:root:i=12 accuracy=tensor(0.0002)
DEBUG:root:i=13 accuracy=tensor(0.0001)
DEBUG:root:i=14 accuracy=tensor(8.1083e-05)
DEBUG:root:i=15 accuracy=tensor(4.7269e-05)
DEBUG:root:i=16 accuracy=tensor(2.7705e-05)
DEBUG:root:i=17 accuracy=tensor(1.6171e-05)
DEBUG:root:i=18 accuracy=tensor(9.4791e-06)
DEBUG:root:i=19 accuracy=tensor(5.4783e-06)
DEBUG:root:i=20 accuracy=tensor(3.2123e-06)
DEBUG:root:i=21 accuracy=tensor(1.8802e-06)
DEBUG:root:i=22 accuracy=tensor(1.1228e-06)
DEBUG:root:i=23 accuracy=tensor(6.3220e-07)
INFO:root:rank=0 pagerank=0.66269850730896 url=4
INFO:root:rank=1 pagerank=0.5217869877815247 url=6
INFO:root:rank=2 pagerank=0.4143447279930115 url=5
INFO:root:rank=3 pagerank=0.2317543923854828 url=2
INFO:root:rank=4 pagerank=0.18590238690376282 url=3
INFO:root:rank=5 pagerank=0.16916769742965698 url=1
```

   Task 1, part 2:
```   
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='corona'
   
   INFO:root:rank=0 pagerank=0.0010038255713880062 url=www.lawfareblog.com/lawfare-podcast-united-nations-and-coronavirus-crisis
INFO:root:rank=1 pagerank=0.0008922845008783042 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
INFO:root:rank=2 pagerank=0.0008077207021415234 url=www.lawfareblog.com/dc-circuits-thoroughly-convincing-decision-al-nashiri
INFO:root:rank=3 pagerank=0.0007039423799142241 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=4 pagerank=0.0006915733683854342 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=5 pagerank=0.0006704507395625114 url=www.lawfareblog.com/israeli-emergency-regulations-location-tracking-coronavirus-carriers
INFO:root:rank=6 pagerank=0.0006625971291214228 url=www.lawfareblog.com/why-congress-conducting-business-usual-face-coronavirus
INFO:root:rank=7 pagerank=0.0006504959892481565 url=www.lawfareblog.com/congressional-homeland-security-committees-seek-ways-support-state-federal-responses-coronavirus
INFO:root:rank=8 pagerank=0.0006362335407175124 url=www.lawfareblog.com/paper-hearing-experts-debate-digital-contact-tracing-and-coronavirus-privacy-concerns
INFO:root:rank=9 pagerank=0.0006125196232460439 url=www.lawfareblog.com/house-subcommittee-voices-concerns-over-us-management-coronavirus

   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='Trump'
   INFO:root:rank=0 pagerank=0.005782715510576963 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=0.005234026350080967 url=www.lawfareblog.com/document-trump-revokes-obama-executive-order-counterterrorism-strike-casualty-reporting
INFO:root:rank=2 pagerank=0.00512984674423933 url=www.lawfareblog.com/trump-administrations-worrying-new-policy-israeli-settlements
INFO:root:rank=3 pagerank=0.004660056438297033 url=www.lawfareblog.com/dc-circuit-overrules-district-courts-due-process-ruling-qasim-v-trump
INFO:root:rank=4 pagerank=0.004593539517372847 url=www.lawfareblog.com/donald-trump-and-politically-weaponized-executive-branch
INFO:root:rank=5 pagerank=0.004307268653064966 url=www.lawfareblog.com/how-trumps-approach-middle-east-ignores-past-future-and-human-condition
INFO:root:rank=6 pagerank=0.004093618597835302 url=www.lawfareblog.com/why-trump-cant-buy-greenland
INFO:root:rank=7 pagerank=0.00375921418890357 url=www.lawfareblog.com/oral-argument-summary-qassim-v-trump
INFO:root:rank=8 pagerank=0.003450991353020072 url=www.lawfareblog.com/dc-circuit-court-denies-trump-rehearing-mazars-case
INFO:root:rank=9 pagerank=0.0034485585056245327 url=www.lawfareblog.com/second-circuit-rules-mazars-must-hand-over-trump-tax-returns-new-york-prosecutors

   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --search_query='Iran'
  
  INFO:root:rank=0 pagerank=0.00512984674423933 url=www.lawfareblog.com/trump-administrations-worrying-new-policy-israeli-settlements
INFO:root:rank=1 pagerank=0.005016941111534834 url=www.lawfareblog.com/update-military-commissions-continued-health-issues-recusal-motion-and-new-cell-al-iraqi
INFO:root:rank=2 pagerank=0.004574750084429979 url=www.lawfareblog.com/praise-presidents-iran-tweets
INFO:root:rank=3 pagerank=0.004417551215738058 url=www.lawfareblog.com/how-us-iran-tensions-could-disrupt-iraqs-fragile-peace
INFO:root:rank=4 pagerank=0.003423787420615554 url=www.lawfareblog.com/france-makes-play-try-foreign-fighters-iraq
INFO:root:rank=5 pagerank=0.0026928838342428207 url=www.lawfareblog.com/cyber-command-operational-update-clarifying-june-2019-iran-operation
INFO:root:rank=6 pagerank=0.0022567661944776773 url=www.lawfareblog.com/document-sens-kaine-and-young-introduce-bill-revoke-iraq-force-authorizations
INFO:root:rank=7 pagerank=0.0019392124377191067 url=www.lawfareblog.com/aborted-iran-strike-fine-line-between-necessity-and-revenge
INFO:root:rank=8 pagerank=0.0018263398669660091 url=www.lawfareblog.com/its-not-only-iraq-and-syria
INFO:root:rank=9 pagerank=0.0017331625567749143 url=www.lawfareblog.com/assessing-aclu-habeas-petition-behalf-unnamed-us-citizen-held-enemy-combatant-iraq
```

   Task 1, part 3:
   ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz
   
   INFO:root:rank=0 pagerank=0.28740572929382324 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=1 pagerank=0.28740572929382324 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=2 pagerank=0.28740572929382324 url=www.lawfareblog.com/masthead
INFO:root:rank=3 pagerank=0.28740572929382324 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
INFO:root:rank=4 pagerank=0.28740572929382324 url=www.lawfareblog.com/subscribe-lawfare
INFO:root:rank=5 pagerank=0.28740572929382324 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
INFO:root:rank=6 pagerank=0.28740572929382324 url=www.lawfareblog.com/documents-related-mueller-investigation
INFO:root:rank=7 pagerank=0.28740572929382324 url=www.lawfareblog.com/our-comments-policy
INFO:root:rank=8 pagerank=0.28740572929382324 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 pagerank=0.28740572929382324 url=www.lawfareblog.com/topics
   ```

   Task 1, part 4:
```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose 
   DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 accuracy=tensor(1.3794)
DEBUG:root:i=1 accuracy=tensor(0.1164)
DEBUG:root:i=2 accuracy=tensor(0.0750)
DEBUG:root:i=3 accuracy=tensor(0.0317)
DEBUG:root:i=4 accuracy=tensor(0.0174)
DEBUG:root:i=5 accuracy=tensor(0.0085)
DEBUG:root:i=6 accuracy=tensor(0.0044)
DEBUG:root:i=7 accuracy=tensor(0.0022)
DEBUG:root:i=8 accuracy=tensor(0.0011)
DEBUG:root:i=9 accuracy=tensor(0.0006)
DEBUG:root:i=10 accuracy=tensor(0.0003)
DEBUG:root:i=11 accuracy=tensor(0.0001)
DEBUG:root:i=12 accuracy=tensor(7.1515e-05)
DEBUG:root:i=13 accuracy=tensor(3.4769e-05)
DEBUG:root:i=14 accuracy=tensor(1.5953e-05)
DEBUG:root:i=15 accuracy=tensor(6.4547e-06)
DEBUG:root:i=16 accuracy=tensor(2.4701e-06)
DEBUG:root:i=17 accuracy=tensor(8.2367e-07)
INFO:root:rank=0 pagerank=0.28740572929382324 url=www.lawfareblog.com/about-lawfare-brief-history-term-and-site
INFO:root:rank=1 pagerank=0.28740572929382324 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=2 pagerank=0.28740572929382324 url=www.lawfareblog.com/masthead
INFO:root:rank=3 pagerank=0.28740572929382324 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
INFO:root:rank=4 pagerank=0.28740572929382324 url=www.lawfareblog.com/subscribe-lawfare
INFO:root:rank=5 pagerank=0.28740572929382324 url=www.lawfareblog.com/litigation-documents-related-appointment-matthew-whitaker-acting-attorney-general
INFO:root:rank=6 pagerank=0.28740572929382324 url=www.lawfareblog.com/documents-related-mueller-investigation
INFO:root:rank=7 pagerank=0.28740572929382324 url=www.lawfareblog.com/our-comments-policy
INFO:root:rank=8 pagerank=0.28740572929382324 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 pagerank=0.28740572929382324 url=www.lawfareblog.com/topics
   
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --alpha=0.99999
   DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 accuracy=tensor(1.3846)
DEBUG:root:i=1 accuracy=tensor(0.0709)
DEBUG:root:i=2 accuracy=tensor(0.0188)
DEBUG:root:i=3 accuracy=tensor(0.0070)
DEBUG:root:i=4 accuracy=tensor(0.0027)
DEBUG:root:i=5 accuracy=tensor(0.0010)
DEBUG:root:i=6 accuracy=tensor(0.0004)
DEBUG:root:i=7 accuracy=tensor(0.0001)
DEBUG:root:i=8 accuracy=tensor(4.8224e-05)
DEBUG:root:i=9 accuracy=tensor(1.7172e-05)
DEBUG:root:i=10 accuracy=tensor(6.1166e-06)
DEBUG:root:i=11 accuracy=tensor(2.1728e-06)
DEBUG:root:i=12 accuracy=tensor(7.7593e-07)
INFO:root:rank=0 pagerank=0.28859299421310425 url=www.lawfareblog.com/snowden-revelations
INFO:root:rank=1 pagerank=0.28859299421310425 url=www.lawfareblog.com/lawfare-job-board
INFO:root:rank=2 pagerank=0.28859299421310425 url=www.lawfareblog.com/documents-related-mueller-investigation
INFO:root:rank=3 pagerank=0.28859299421310425 url=www.lawfareblog.com/litigation-documents-resources-related-travel-ban
INFO:root:rank=4 pagerank=0.28859299421310425 url=www.lawfareblog.com/subscribe-lawfare
INFO:root:rank=5 pagerank=0.28859299421310425 url=www.lawfareblog.com/topics
INFO:root:rank=6 pagerank=0.28859299421310425 url=www.lawfareblog.com/masthead
INFO:root:rank=7 pagerank=0.28859299421310425 url=www.lawfareblog.com/our-comments-policy
INFO:root:rank=8 pagerank=0.28859299421310425 url=www.lawfareblog.com/upcoming-events
INFO:root:rank=9 pagerank=0.28859299421310425 url=www.lawfareb

   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2
   DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 accuracy=tensor(1.2610)
DEBUG:root:i=1 accuracy=tensor(0.4986)
DEBUG:root:i=2 accuracy=tensor(0.1342)
DEBUG:root:i=3 accuracy=tensor(0.0692)
DEBUG:root:i=4 accuracy=tensor(0.0234)
DEBUG:root:i=5 accuracy=tensor(0.0102)
DEBUG:root:i=6 accuracy=tensor(0.0049)
DEBUG:root:i=7 accuracy=tensor(0.0023)
DEBUG:root:i=8 accuracy=tensor(0.0011)
DEBUG:root:i=9 accuracy=tensor(0.0005)
DEBUG:root:i=10 accuracy=tensor(0.0003)
DEBUG:root:i=11 accuracy=tensor(0.0001)
DEBUG:root:i=12 accuracy=tensor(8.2266e-05)
DEBUG:root:i=13 accuracy=tensor(4.8133e-05)
DEBUG:root:i=14 accuracy=tensor(2.8801e-05)
DEBUG:root:i=15 accuracy=tensor(1.7420e-05)
DEBUG:root:i=16 accuracy=tensor(1.0540e-05)
DEBUG:root:i=17 accuracy=tensor(6.3960e-06)
DEBUG:root:i=18 accuracy=tensor(3.8482e-06)
DEBUG:root:i=19 accuracy=tensor(2.2987e-06)
DEBUG:root:i=20 accuracy=tensor(1.3677e-06)
DEBUG:root:i=21 accuracy=tensor(8.1541e-07)
INFO:root:rank=0 pagerank=0.34696561098098755 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=1 pagerank=0.2952174246311188 url=www.lawfareblog.com/livestream-nov-21-impeachment-hearings-0
INFO:root:rank=2 pagerank=0.29040172696113586 url=www.lawfareblog.com/opening-statement-david-holmes
INFO:root:rank=3 pagerank=0.1517927199602127 url=www.lawfareblog.com/lawfare-podcast-ben-nimmo-whack-mole-game-disinformation
INFO:root:rank=4 pagerank=0.1509954184293747 url=www.lawfareblog.com/todays-headlines-and-commentary-1964
INFO:root:rank=5 pagerank=0.1509954184293747 url=www.lawfareblog.com/todays-headlines-and-commentary-1963
INFO:root:rank=6 pagerank=0.15071777999401093 url=www.lawfareblog.com/lawfare-podcast-week-was-impeachment
INFO:root:rank=7 pagerank=0.14957697689533234 url=www.lawfareblog.com/todays-headlines-and-commentary-1962
INFO:root:rank=8 pagerank=0.14367210865020752 url=www.lawfareblog.com/cyberlaw-podcast-mistrusting-google
INFO:root:rank=9 pagerank=0.14240293204784393 url=www.lawfareblog.com/lawfare-podcast-bonus-edition-gordon-sondland-vs-committee-no-bull

   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --verbose --filter_ratio=0.2 --alpha=0.99999
   DEBUG:root:computing indices
DEBUG:root:computing values
DEBUG:root:i=0 accuracy=tensor(1.2828)
DEBUG:root:i=1 accuracy=tensor(0.5696)
DEBUG:root:i=2 accuracy=tensor(0.3830)
DEBUG:root:i=3 accuracy=tensor(0.2174)
DEBUG:root:i=4 accuracy=tensor(0.1405)
DEBUG:root:i=5 accuracy=tensor(0.1085)
DEBUG:root:i=6 accuracy=tensor(0.0928)
DEBUG:root:i=7 accuracy=tensor(0.0823)
DEBUG:root:i=8 accuracy=tensor(0.0734)
DEBUG:root:i=9 accuracy=tensor(0.0656)
DEBUG:root:i=10 accuracy=tensor(0.0591)
DEBUG:root:i=11 accuracy=tensor(0.0542)
DEBUG:root:i=12 accuracy=tensor(0.0511)
DEBUG:root:i=13 accuracy=tensor(0.0500)
DEBUG:root:i=14 accuracy=tensor(0.0506)
DEBUG:root:i=15 accuracy=tensor(0.0525)
DEBUG:root:i=16 accuracy=tensor(0.0552)
DEBUG:root:i=17 accuracy=tensor(0.0580)
DEBUG:root:i=18 accuracy=tensor(0.0606)
DEBUG:root:i=19 accuracy=tensor(0.0625)
DEBUG:root:i=20 accuracy=tensor(0.0635)
DEBUG:root:i=21 accuracy=tensor(0.0634)
DEBUG:root:i=22 accuracy=tensor(0.0623)
DEBUG:root:i=23 accuracy=tensor(0.0604)
DEBUG:root:i=24 accuracy=tensor(0.0577)
DEBUG:root:i=25 accuracy=tensor(0.0545)
DEBUG:root:i=26 accuracy=tensor(0.0509)
DEBUG:root:i=27 accuracy=tensor(0.0473)
DEBUG:root:i=28 accuracy=tensor(0.0436)
DEBUG:root:i=29 accuracy=tensor(0.0400)
DEBUG:root:i=30 accuracy=tensor(0.0366)
DEBUG:root:i=31 accuracy=tensor(0.0334)
DEBUG:root:i=32 accuracy=tensor(0.0305)
DEBUG:root:i=33 accuracy=tensor(0.0278)
DEBUG:root:i=34 accuracy=tensor(0.0254)
DEBUG:root:i=35 accuracy=tensor(0.0231)
DEBUG:root:i=36 accuracy=tensor(0.0212)
DEBUG:root:i=37 accuracy=tensor(0.0194)
DEBUG:root:i=38 accuracy=tensor(0.0177)
DEBUG:root:i=39 accuracy=tensor(0.0163)
DEBUG:root:i=40 accuracy=tensor(0.0150)
DEBUG:root:i=41 accuracy=tensor(0.0138)
DEBUG:root:i=42 accuracy=tensor(0.0127)
DEBUG:root:i=43 accuracy=tensor(0.0118)
DEBUG:root:i=44 accuracy=tensor(0.0109)
DEBUG:root:i=45 accuracy=tensor(0.0102)
DEBUG:root:i=46 accuracy=tensor(0.0094)
DEBUG:root:i=47 accuracy=tensor(0.0088)
DEBUG:root:i=48 accuracy=tensor(0.0082)
DEBUG:root:i=49 accuracy=tensor(0.0077)
DEBUG:root:i=50 accuracy=tensor(0.0072)
DEBUG:root:i=51 accuracy=tensor(0.0068)
DEBUG:root:i=52 accuracy=tensor(0.0064)
DEBUG:root:i=53 accuracy=tensor(0.0060)
DEBUG:root:i=54 accuracy=tensor(0.0057)
DEBUG:root:i=55 accuracy=tensor(0.0053)
DEBUG:root:i=56 accuracy=tensor(0.0051)
DEBUG:root:i=57 accuracy=tensor(0.0048)
DEBUG:root:i=58 accuracy=tensor(0.0045)
DEBUG:root:i=59 accuracy=tensor(0.0043)
DEBUG:root:i=60 accuracy=tensor(0.0041)
DEBUG:root:i=61 accuracy=tensor(0.0039)
DEBUG:root:i=62 accuracy=tensor(0.0037)
DEBUG:root:i=63 accuracy=tensor(0.0036)
DEBUG:root:i=64 accuracy=tensor(0.0034)
DEBUG:root:i=65 accuracy=tensor(0.0033)
DEBUG:root:i=66 accuracy=tensor(0.0031)
DEBUG:root:i=67 accuracy=tensor(0.0030)
DEBUG:root:i=68 accuracy=tensor(0.0029)
DEBUG:root:i=69 accuracy=tensor(0.0028)
DEBUG:root:i=70 accuracy=tensor(0.0027)
DEBUG:root:i=71 accuracy=tensor(0.0026)
DEBUG:root:i=72 accuracy=tensor(0.0025)
DEBUG:root:i=73 accuracy=tensor(0.0024)
DEBUG:root:i=74 accuracy=tensor(0.0023)
DEBUG:root:i=75 accuracy=tensor(0.0022)
DEBUG:root:i=76 accuracy=tensor(0.0021)
DEBUG:root:i=77 accuracy=tensor(0.0021)
DEBUG:root:i=78 accuracy=tensor(0.0020)
DEBUG:root:i=79 accuracy=tensor(0.0019)
DEBUG:root:i=80 accuracy=tensor(0.0019)
DEBUG:root:i=81 accuracy=tensor(0.0018)
DEBUG:root:i=82 accuracy=tensor(0.0018)
DEBUG:root:i=83 accuracy=tensor(0.0017)
DEBUG:root:i=84 accuracy=tensor(0.0017)
DEBUG:root:i=85 accuracy=tensor(0.0016)
DEBUG:root:i=86 accuracy=tensor(0.0016)
DEBUG:root:i=87 accuracy=tensor(0.0015)
DEBUG:root:i=88 accuracy=tensor(0.0015)
DEBUG:root:i=89 accuracy=tensor(0.0014)
DEBUG:root:i=90 accuracy=tensor(0.0014)
DEBUG:root:i=91 accuracy=tensor(0.0014)
DEBUG:root:i=92 accuracy=tensor(0.0013)
DEBUG:root:i=93 accuracy=tensor(0.0013)
DEBUG:root:i=94 accuracy=tensor(0.0013)
DEBUG:root:i=95 accuracy=tensor(0.0012)
DEBUG:root:i=96 accuracy=tensor(0.0012)
DEBUG:root:i=97 accuracy=tensor(0.0012)
DEBUG:root:i=98 accuracy=tensor(0.0011)
DEBUG:root:i=99 accuracy=tensor(0.0011)
DEBUG:root:i=100 accuracy=tensor(0.0011)
DEBUG:root:i=101 accuracy=tensor(0.0011)
DEBUG:root:i=102 accuracy=tensor(0.0010)
DEBUG:root:i=103 accuracy=tensor(0.0010)
DEBUG:root:i=104 accuracy=tensor(0.0010)
DEBUG:root:i=105 accuracy=tensor(0.0010)
DEBUG:root:i=106 accuracy=tensor(0.0010)
DEBUG:root:i=107 accuracy=tensor(0.0009)
DEBUG:root:i=108 accuracy=tensor(0.0009)
DEBUG:root:i=109 accuracy=tensor(0.0009)
DEBUG:root:i=110 accuracy=tensor(0.0009)
DEBUG:root:i=111 accuracy=tensor(0.0009)
DEBUG:root:i=112 accuracy=tensor(0.0008)
DEBUG:root:i=113 accuracy=tensor(0.0008)
DEBUG:root:i=114 accuracy=tensor(0.0008)
DEBUG:root:i=115 accuracy=tensor(0.0008)
DEBUG:root:i=116 accuracy=tensor(0.0008)
DEBUG:root:i=117 accuracy=tensor(0.0008)
DEBUG:root:i=118 accuracy=tensor(0.0008)
DEBUG:root:i=119 accuracy=tensor(0.0007)
DEBUG:root:i=120 accuracy=tensor(0.0007)
DEBUG:root:i=121 accuracy=tensor(0.0007)
DEBUG:root:i=122 accuracy=tensor(0.0007)
DEBUG:root:i=123 accuracy=tensor(0.0007)
DEBUG:root:i=124 accuracy=tensor(0.0007)
DEBUG:root:i=125 accuracy=tensor(0.0007)
DEBUG:root:i=126 accuracy=tensor(0.0007)
DEBUG:root:i=127 accuracy=tensor(0.0006)
DEBUG:root:i=128 accuracy=tensor(0.0006)
DEBUG:root:i=129 accuracy=tensor(0.0006)
DEBUG:root:i=130 accuracy=tensor(0.0006)
DEBUG:root:i=131 accuracy=tensor(0.0006)
DEBUG:root:i=132 accuracy=tensor(0.0006)
DEBUG:root:i=133 accuracy=tensor(0.0006)
DEBUG:root:i=134 accuracy=tensor(0.0006)
DEBUG:root:i=135 accuracy=tensor(0.0006)
DEBUG:root:i=136 accuracy=tensor(0.0006)
DEBUG:root:i=137 accuracy=tensor(0.0005)
DEBUG:root:i=138 accuracy=tensor(0.0005)
DEBUG:root:i=139 accuracy=tensor(0.0005)
DEBUG:root:i=140 accuracy=tensor(0.0005)
DEBUG:root:i=141 accuracy=tensor(0.0005)
DEBUG:root:i=142 accuracy=tensor(0.0005)
DEBUG:root:i=143 accuracy=tensor(0.0005)
DEBUG:root:i=144 accuracy=tensor(0.0005)
DEBUG:root:i=145 accuracy=tensor(0.0005)
DEBUG:root:i=146 accuracy=tensor(0.0005)
DEBUG:root:i=147 accuracy=tensor(0.0005)
DEBUG:root:i=148 accuracy=tensor(0.0005)
DEBUG:root:i=149 accuracy=tensor(0.0005)
DEBUG:root:i=150 accuracy=tensor(0.0004)
DEBUG:root:i=151 accuracy=tensor(0.0004)
DEBUG:root:i=152 accuracy=tensor(0.0004)
DEBUG:root:i=153 accuracy=tensor(0.0004)
DEBUG:root:i=154 accuracy=tensor(0.0004)
DEBUG:root:i=155 accuracy=tensor(0.0004)
DEBUG:root:i=156 accuracy=tensor(0.0004)
DEBUG:root:i=157 accuracy=tensor(0.0004)
DEBUG:root:i=158 accuracy=tensor(0.0004)
DEBUG:root:i=159 accuracy=tensor(0.0004)
DEBUG:root:i=160 accuracy=tensor(0.0004)
DEBUG:root:i=161 accuracy=tensor(0.0004)
DEBUG:root:i=162 accuracy=tensor(0.0004)
DEBUG:root:i=163 accuracy=tensor(0.0004)
DEBUG:root:i=164 accuracy=tensor(0.0004)
DEBUG:root:i=165 accuracy=tensor(0.0004)
DEBUG:root:i=166 accuracy=tensor(0.0004)
DEBUG:root:i=167 accuracy=tensor(0.0003)
DEBUG:root:i=168 accuracy=tensor(0.0003)
DEBUG:root:i=169 accuracy=tensor(0.0003)
DEBUG:root:i=170 accuracy=tensor(0.0003)
DEBUG:root:i=171 accuracy=tensor(0.0003)
DEBUG:root:i=172 accuracy=tensor(0.0003)
DEBUG:root:i=173 accuracy=tensor(0.0003)
DEBUG:root:i=174 accuracy=tensor(0.0003)
DEBUG:root:i=175 accuracy=tensor(0.0003)
DEBUG:root:i=176 accuracy=tensor(0.0003)
DEBUG:root:i=177 accuracy=tensor(0.0003)
DEBUG:root:i=178 accuracy=tensor(0.0003)
DEBUG:root:i=179 accuracy=tensor(0.0003)
DEBUG:root:i=180 accuracy=tensor(0.0003)
DEBUG:root:i=181 accuracy=tensor(0.0003)
DEBUG:root:i=182 accuracy=tensor(0.0003)
DEBUG:root:i=183 accuracy=tensor(0.0003)
DEBUG:root:i=184 accuracy=tensor(0.0003)
DEBUG:root:i=185 accuracy=tensor(0.0003)
DEBUG:root:i=186 accuracy=tensor(0.0003)
DEBUG:root:i=187 accuracy=tensor(0.0003)
DEBUG:root:i=188 accuracy=tensor(0.0003)
DEBUG:root:i=189 accuracy=tensor(0.0003)
DEBUG:root:i=190 accuracy=tensor(0.0003)
DEBUG:root:i=191 accuracy=tensor(0.0003)
DEBUG:root:i=192 accuracy=tensor(0.0002)
DEBUG:root:i=193 accuracy=tensor(0.0002)
DEBUG:root:i=194 accuracy=tensor(0.0002)
DEBUG:root:i=195 accuracy=tensor(0.0002)
DEBUG:root:i=196 accuracy=tensor(0.0002)
DEBUG:root:i=197 accuracy=tensor(0.0002)
DEBUG:root:i=198 accuracy=tensor(0.0002)
DEBUG:root:i=199 accuracy=tensor(0.0002)
DEBUG:root:i=200 accuracy=tensor(0.0002)
DEBUG:root:i=201 accuracy=tensor(0.0002)
DEBUG:root:i=202 accuracy=tensor(0.0002)
DEBUG:root:i=203 accuracy=tensor(0.0002)
DEBUG:root:i=204 accuracy=tensor(0.0002)
DEBUG:root:i=205 accuracy=tensor(0.0002)
DEBUG:root:i=206 accuracy=tensor(0.0002)
DEBUG:root:i=207 accuracy=tensor(0.0002)
DEBUG:root:i=208 accuracy=tensor(0.0002)
DEBUG:root:i=209 accuracy=tensor(0.0002)
DEBUG:root:i=210 accuracy=tensor(0.0002)
DEBUG:root:i=211 accuracy=tensor(0.0002)
DEBUG:root:i=212 accuracy=tensor(0.0002)
DEBUG:root:i=213 accuracy=tensor(0.0002)
DEBUG:root:i=214 accuracy=tensor(0.0002)
DEBUG:root:i=215 accuracy=tensor(0.0002)
DEBUG:root:i=216 accuracy=tensor(0.0002)
DEBUG:root:i=217 accuracy=tensor(0.0002)
DEBUG:root:i=218 accuracy=tensor(0.0002)
DEBUG:root:i=219 accuracy=tensor(0.0002)
DEBUG:root:i=220 accuracy=tensor(0.0002)
DEBUG:root:i=221 accuracy=tensor(0.0002)
DEBUG:root:i=222 accuracy=tensor(0.0002)
DEBUG:root:i=223 accuracy=tensor(0.0002)
DEBUG:root:i=224 accuracy=tensor(0.0002)
DEBUG:root:i=225 accuracy=tensor(0.0002)
DEBUG:root:i=226 accuracy=tensor(0.0002)
DEBUG:root:i=227 accuracy=tensor(0.0002)
DEBUG:root:i=228 accuracy=tensor(0.0002)
DEBUG:root:i=229 accuracy=tensor(0.0002)
DEBUG:root:i=230 accuracy=tensor(0.0002)
DEBUG:root:i=231 accuracy=tensor(0.0002)
DEBUG:root:i=232 accuracy=tensor(0.0002)
DEBUG:root:i=233 accuracy=tensor(0.0001)
DEBUG:root:i=234 accuracy=tensor(0.0001)
DEBUG:root:i=235 accuracy=tensor(0.0001)
DEBUG:root:i=236 accuracy=tensor(0.0001)
DEBUG:root:i=237 accuracy=tensor(0.0001)
DEBUG:root:i=238 accuracy=tensor(0.0001)
DEBUG:root:i=239 accuracy=tensor(0.0001)
DEBUG:root:i=240 accuracy=tensor(0.0001)
DEBUG:root:i=241 accuracy=tensor(0.0001)
DEBUG:root:i=242 accuracy=tensor(0.0001)
DEBUG:root:i=243 accuracy=tensor(0.0001)
DEBUG:root:i=244 accuracy=tensor(0.0001)
DEBUG:root:i=245 accuracy=tensor(0.0001)
DEBUG:root:i=246 accuracy=tensor(0.0001)
DEBUG:root:i=247 accuracy=tensor(0.0001)
DEBUG:root:i=248 accuracy=tensor(0.0001)
DEBUG:root:i=249 accuracy=tensor(0.0001)
DEBUG:root:i=250 accuracy=tensor(0.0001)
DEBUG:root:i=251 accuracy=tensor(0.0001)
DEBUG:root:i=252 accuracy=tensor(0.0001)
DEBUG:root:i=253 accuracy=tensor(0.0001)
DEBUG:root:i=254 accuracy=tensor(0.0001)
DEBUG:root:i=255 accuracy=tensor(0.0001)
DEBUG:root:i=256 accuracy=tensor(0.0001)
DEBUG:root:i=257 accuracy=tensor(0.0001)
DEBUG:root:i=258 accuracy=tensor(0.0001)
DEBUG:root:i=259 accuracy=tensor(0.0001)
DEBUG:root:i=260 accuracy=tensor(0.0001)
DEBUG:root:i=261 accuracy=tensor(0.0001)
DEBUG:root:i=262 accuracy=tensor(0.0001)
DEBUG:root:i=263 accuracy=tensor(0.0001)
DEBUG:root:i=264 accuracy=tensor(0.0001)
DEBUG:root:i=265 accuracy=tensor(0.0001)
DEBUG:root:i=266 accuracy=tensor(0.0001)
DEBUG:root:i=267 accuracy=tensor(0.0001)
DEBUG:root:i=268 accuracy=tensor(9.8927e-05)
DEBUG:root:i=269 accuracy=tensor(9.7781e-05)
DEBUG:root:i=270 accuracy=tensor(9.6649e-05)
DEBUG:root:i=271 accuracy=tensor(9.5529e-05)
DEBUG:root:i=272 accuracy=tensor(9.4425e-05)
DEBUG:root:i=273 accuracy=tensor(9.3334e-05)
DEBUG:root:i=274 accuracy=tensor(9.2255e-05)
DEBUG:root:i=275 accuracy=tensor(9.1190e-05)
DEBUG:root:i=276 accuracy=tensor(9.0137e-05)
DEBUG:root:i=277 accuracy=tensor(8.9100e-05)
DEBUG:root:i=278 accuracy=tensor(8.8073e-05)
DEBUG:root:i=279 accuracy=tensor(8.7059e-05)
DEBUG:root:i=280 accuracy=tensor(8.6058e-05)
DEBUG:root:i=281 accuracy=tensor(8.5069e-05)
DEBUG:root:i=282 accuracy=tensor(8.4091e-05)
DEBUG:root:i=283 accuracy=tensor(8.3126e-05)
DEBUG:root:i=284 accuracy=tensor(8.2173e-05)
DEBUG:root:i=285 accuracy=tensor(8.1232e-05)
DEBUG:root:i=286 accuracy=tensor(8.0300e-05)
DEBUG:root:i=287 accuracy=tensor(7.9381e-05)
DEBUG:root:i=288 accuracy=tensor(7.8474e-05)
DEBUG:root:i=289 accuracy=tensor(7.7576e-05)
DEBUG:root:i=290 accuracy=tensor(7.6690e-05)
DEBUG:root:i=291 accuracy=tensor(7.5814e-05)
DEBUG:root:i=292 accuracy=tensor(7.4950e-05)
DEBUG:root:i=293 accuracy=tensor(7.4094e-05)
DEBUG:root:i=294 accuracy=tensor(7.3250e-05)
DEBUG:root:i=295 accuracy=tensor(7.2415e-05)
DEBUG:root:i=296 accuracy=tensor(7.1591e-05)
DEBUG:root:i=297 accuracy=tensor(7.0776e-05)
DEBUG:root:i=298 accuracy=tensor(6.9972e-05)
DEBUG:root:i=299 accuracy=tensor(6.9177e-05)
DEBUG:root:i=300 accuracy=tensor(6.8391e-05)
DEBUG:root:i=301 accuracy=tensor(6.7615e-05)
DEBUG:root:i=302 accuracy=tensor(6.6847e-05)
DEBUG:root:i=303 accuracy=tensor(6.6088e-05)
DEBUG:root:i=304 accuracy=tensor(6.5340e-05)
DEBUG:root:i=305 accuracy=tensor(6.4600e-05)
DEBUG:root:i=306 accuracy=tensor(6.3868e-05)
DEBUG:root:i=307 accuracy=tensor(6.3145e-05)
DEBUG:root:i=308 accuracy=tensor(6.2432e-05)
DEBUG:root:i=309 accuracy=tensor(6.1724e-05)
DEBUG:root:i=310 accuracy=tensor(6.1029e-05)
DEBUG:root:i=311 accuracy=tensor(6.0339e-05)
DEBUG:root:i=312 accuracy=tensor(5.9659e-05)
DEBUG:root:i=313 accuracy=tensor(5.8985e-05)
DEBUG:root:i=314 accuracy=tensor(5.8319e-05)
DEBUG:root:i=315 accuracy=tensor(5.7661e-05)
DEBUG:root:i=316 accuracy=tensor(5.7009e-05)
DEBUG:root:i=317 accuracy=tensor(5.6369e-05)
DEBUG:root:i=318 accuracy=tensor(5.5733e-05)
DEBUG:root:i=319 accuracy=tensor(5.5106e-05)
DEBUG:root:i=320 accuracy=tensor(5.4486e-05)
DEBUG:root:i=321 accuracy=tensor(5.3874e-05)
DEBUG:root:i=322 accuracy=tensor(5.3268e-05)
DEBUG:root:i=323 accuracy=tensor(5.2670e-05)
DEBUG:root:i=324 accuracy=tensor(5.2078e-05)
DEBUG:root:i=325 accuracy=tensor(5.1494e-05)
DEBUG:root:i=326 accuracy=tensor(5.0917e-05)
DEBUG:root:i=327 accuracy=tensor(5.0346e-05)
DEBUG:root:i=328 accuracy=tensor(4.9781e-05)
DEBUG:root:i=329 accuracy=tensor(4.9222e-05)
DEBUG:root:i=330 accuracy=tensor(4.8669e-05)
DEBUG:root:i=331 accuracy=tensor(4.8124e-05)
DEBUG:root:i=332 accuracy=tensor(4.7587e-05)
DEBUG:root:i=333 accuracy=tensor(4.7056e-05)
DEBUG:root:i=334 accuracy=tensor(4.6528e-05)
DEBUG:root:i=335 accuracy=tensor(4.6007e-05)
DEBUG:root:i=336 accuracy=tensor(4.5492e-05)
DEBUG:root:i=337 accuracy=tensor(4.4985e-05)
DEBUG:root:i=338 accuracy=tensor(4.4483e-05)
DEBUG:root:i=339 accuracy=tensor(4.3985e-05)
DEBUG:root:i=340 accuracy=tensor(4.3494e-05)
DEBUG:root:i=341 accuracy=tensor(4.3010e-05)
DEBUG:root:i=342 accuracy=tensor(4.2531e-05)
DEBUG:root:i=343 accuracy=tensor(4.2054e-05)
DEBUG:root:i=344 accuracy=tensor(4.1587e-05)
DEBUG:root:i=345 accuracy=tensor(4.1126e-05)
DEBUG:root:i=346 accuracy=tensor(4.0665e-05)
DEBUG:root:i=347 accuracy=tensor(4.0215e-05)
DEBUG:root:i=348 accuracy=tensor(3.9765e-05)
DEBUG:root:i=349 accuracy=tensor(3.9323e-05)
DEBUG:root:i=350 accuracy=tensor(3.8887e-05)
DEBUG:root:i=351 accuracy=tensor(3.8453e-05)
DEBUG:root:i=352 accuracy=tensor(3.8027e-05)
DEBUG:root:i=353 accuracy=tensor(3.7604e-05)
DEBUG:root:i=354 accuracy=tensor(3.7186e-05)
DEBUG:root:i=355 accuracy=tensor(3.6771e-05)
DEBUG:root:i=356 accuracy=tensor(3.6364e-05)
DEBUG:root:i=357 accuracy=tensor(3.5962e-05)
DEBUG:root:i=358 accuracy=tensor(3.5562e-05)
DEBUG:root:i=359 accuracy=tensor(3.5167e-05)
DEBUG:root:i=360 accuracy=tensor(3.4777e-05)
DEBUG:root:i=361 accuracy=tensor(3.4392e-05)
DEBUG:root:i=362 accuracy=tensor(3.4011e-05)
DEBUG:root:i=363 accuracy=tensor(3.3633e-05)
DEBUG:root:i=364 accuracy=tensor(3.3262e-05)
DEBUG:root:i=365 accuracy=tensor(3.2893e-05)
DEBUG:root:i=366 accuracy=tensor(3.2529e-05)
DEBUG:root:i=367 accuracy=tensor(3.2169e-05)
DEBUG:root:i=368 accuracy=tensor(3.1813e-05)
DEBUG:root:i=369 accuracy=tensor(3.1460e-05)
DEBUG:root:i=370 accuracy=tensor(3.1113e-05)
DEBUG:root:i=371 accuracy=tensor(3.0769e-05)
DEBUG:root:i=372 accuracy=tensor(3.0429e-05)
DEBUG:root:i=373 accuracy=tensor(3.0092e-05)
DEBUG:root:i=374 accuracy=tensor(2.9760e-05)
DEBUG:root:i=375 accuracy=tensor(2.9431e-05)
DEBUG:root:i=376 accuracy=tensor(2.9105e-05)
DEBUG:root:i=377 accuracy=tensor(2.8784e-05)
DEBUG:root:i=378 accuracy=tensor(2.8469e-05)
DEBUG:root:i=379 accuracy=tensor(2.8153e-05)
DEBUG:root:i=380 accuracy=tensor(2.7842e-05)
DEBUG:root:i=381 accuracy=tensor(2.7534e-05)
DEBUG:root:i=382 accuracy=tensor(2.7231e-05)
DEBUG:root:i=383 accuracy=tensor(2.6931e-05)
DEBUG:root:i=384 accuracy=tensor(2.6634e-05)
DEBUG:root:i=385 accuracy=tensor(2.6340e-05)
DEBUG:root:i=386 accuracy=tensor(2.6051e-05)
DEBUG:root:i=387 accuracy=tensor(2.5763e-05)
DEBUG:root:i=388 accuracy=tensor(2.5479e-05)
DEBUG:root:i=389 accuracy=tensor(2.5199e-05)
DEBUG:root:i=390 accuracy=tensor(2.4921e-05)
DEBUG:root:i=391 accuracy=tensor(2.4648e-05)
DEBUG:root:i=392 accuracy=tensor(2.4375e-05)
DEBUG:root:i=393 accuracy=tensor(2.4107e-05)
DEBUG:root:i=394 accuracy=tensor(2.3843e-05)
DEBUG:root:i=395 accuracy=tensor(2.3581e-05)
DEBUG:root:i=396 accuracy=tensor(2.3322e-05)
DEBUG:root:i=397 accuracy=tensor(2.3064e-05)
DEBUG:root:i=398 accuracy=tensor(2.2812e-05)
DEBUG:root:i=399 accuracy=tensor(2.2560e-05)
DEBUG:root:i=400 accuracy=tensor(2.2313e-05)
DEBUG:root:i=401 accuracy=tensor(2.2068e-05)
DEBUG:root:i=402 accuracy=tensor(2.1824e-05)
DEBUG:root:i=403 accuracy=tensor(2.1584e-05)
DEBUG:root:i=404 accuracy=tensor(2.1348e-05)
DEBUG:root:i=405 accuracy=tensor(2.1114e-05)
DEBUG:root:i=406 accuracy=tensor(2.0881e-05)
DEBUG:root:i=407 accuracy=tensor(2.0652e-05)
DEBUG:root:i=408 accuracy=tensor(2.0426e-05)
DEBUG:root:i=409 accuracy=tensor(2.0202e-05)
DEBUG:root:i=410 accuracy=tensor(1.9980e-05)
DEBUG:root:i=411 accuracy=tensor(1.9762e-05)
DEBUG:root:i=412 accuracy=tensor(1.9545e-05)
DEBUG:root:i=413 accuracy=tensor(1.9330e-05)
DEBUG:root:i=414 accuracy=tensor(1.9119e-05)
DEBUG:root:i=415 accuracy=tensor(1.8909e-05)
DEBUG:root:i=416 accuracy=tensor(1.8701e-05)
DEBUG:root:i=417 accuracy=tensor(1.8496e-05)
DEBUG:root:i=418 accuracy=tensor(1.8295e-05)
DEBUG:root:i=419 accuracy=tensor(1.8094e-05)
DEBUG:root:i=420 accuracy=tensor(1.7895e-05)
DEBUG:root:i=421 accuracy=tensor(1.7699e-05)
DEBUG:root:i=422 accuracy=tensor(1.7505e-05)
DEBUG:root:i=423 accuracy=tensor(1.7312e-05)
DEBUG:root:i=424 accuracy=tensor(1.7125e-05)
DEBUG:root:i=425 accuracy=tensor(1.6936e-05)
DEBUG:root:i=426 accuracy=tensor(1.6751e-05)
DEBUG:root:i=427 accuracy=tensor(1.6569e-05)
DEBUG:root:i=428 accuracy=tensor(1.6388e-05)
DEBUG:root:i=429 accuracy=tensor(1.6208e-05)
DEBUG:root:i=430 accuracy=tensor(1.6030e-05)
DEBUG:root:i=431 accuracy=tensor(1.5855e-05)
DEBUG:root:i=432 accuracy=tensor(1.5682e-05)
DEBUG:root:i=433 accuracy=tensor(1.5511e-05)
DEBUG:root:i=434 accuracy=tensor(1.5341e-05)
DEBUG:root:i=435 accuracy=tensor(1.5173e-05)
DEBUG:root:i=436 accuracy=tensor(1.5007e-05)
DEBUG:root:i=437 accuracy=tensor(1.4844e-05)
DEBUG:root:i=438 accuracy=tensor(1.4681e-05)
DEBUG:root:i=439 accuracy=tensor(1.4520e-05)
DEBUG:root:i=440 accuracy=tensor(1.4362e-05)
DEBUG:root:i=441 accuracy=tensor(1.4206e-05)
DEBUG:root:i=442 accuracy=tensor(1.4048e-05)
DEBUG:root:i=443 accuracy=tensor(1.3895e-05)
DEBUG:root:i=444 accuracy=tensor(1.3744e-05)
DEBUG:root:i=445 accuracy=tensor(1.3592e-05)
DEBUG:root:i=446 accuracy=tensor(1.3446e-05)
DEBUG:root:i=447 accuracy=tensor(1.3299e-05)
DEBUG:root:i=448 accuracy=tensor(1.3154e-05)
DEBUG:root:i=449 accuracy=tensor(1.3010e-05)
DEBUG:root:i=450 accuracy=tensor(1.2868e-05)
DEBUG:root:i=451 accuracy=tensor(1.2727e-05)
DEBUG:root:i=452 accuracy=tensor(1.2588e-05)
DEBUG:root:i=453 accuracy=tensor(1.2450e-05)
DEBUG:root:i=454 accuracy=tensor(1.2314e-05)
DEBUG:root:i=455 accuracy=tensor(1.2182e-05)
DEBUG:root:i=456 accuracy=tensor(1.2048e-05)
DEBUG:root:i=457 accuracy=tensor(1.1916e-05)
DEBUG:root:i=458 accuracy=tensor(1.1785e-05)
DEBUG:root:i=459 accuracy=tensor(1.1656e-05)
DEBUG:root:i=460 accuracy=tensor(1.1530e-05)
DEBUG:root:i=461 accuracy=tensor(1.1404e-05)
DEBUG:root:i=462 accuracy=tensor(1.1280e-05)
DEBUG:root:i=463 accuracy=tensor(1.1158e-05)
DEBUG:root:i=464 accuracy=tensor(1.1036e-05)
DEBUG:root:i=465 accuracy=tensor(1.0915e-05)
DEBUG:root:i=466 accuracy=tensor(1.0796e-05)
DEBUG:root:i=467 accuracy=tensor(1.0679e-05)
DEBUG:root:i=468 accuracy=tensor(1.0562e-05)
DEBUG:root:i=469 accuracy=tensor(1.0447e-05)
DEBUG:root:i=470 accuracy=tensor(1.0333e-05)
DEBUG:root:i=471 accuracy=tensor(1.0220e-05)
DEBUG:root:i=472 accuracy=tensor(1.0110e-05)
DEBUG:root:i=473 accuracy=tensor(9.9984e-06)
DEBUG:root:i=474 accuracy=tensor(9.8900e-06)
DEBUG:root:i=475 accuracy=tensor(9.7819e-06)
DEBUG:root:i=476 accuracy=tensor(9.6747e-06)
DEBUG:root:i=477 accuracy=tensor(9.5700e-06)
DEBUG:root:i=478 accuracy=tensor(9.4659e-06)
DEBUG:root:i=479 accuracy=tensor(9.3635e-06)
DEBUG:root:i=480 accuracy=tensor(9.2610e-06)
DEBUG:root:i=481 accuracy=tensor(9.1618e-06)
DEBUG:root:i=482 accuracy=tensor(9.0608e-06)
DEBUG:root:i=483 accuracy=tensor(8.9631e-06)
DEBUG:root:i=484 accuracy=tensor(8.8642e-06)
DEBUG:root:i=485 accuracy=tensor(8.7691e-06)
DEBUG:root:i=486 accuracy=tensor(8.6738e-06)
DEBUG:root:i=487 accuracy=tensor(8.5789e-06)
DEBUG:root:i=488 accuracy=tensor(8.4852e-06)
DEBUG:root:i=489 accuracy=tensor(8.3937e-06)
DEBUG:root:i=490 accuracy=tensor(8.3026e-06)
DEBUG:root:i=491 accuracy=tensor(8.2113e-06)
DEBUG:root:i=492 accuracy=tensor(8.1227e-06)
DEBUG:root:i=493 accuracy=tensor(8.0347e-06)
DEBUG:root:i=494 accuracy=tensor(7.9461e-06)
DEBUG:root:i=495 accuracy=tensor(7.8610e-06)
DEBUG:root:i=496 accuracy=tensor(7.7748e-06)
DEBUG:root:i=497 accuracy=tensor(7.6900e-06)
DEBUG:root:i=498 accuracy=tensor(7.6069e-06)
DEBUG:root:i=499 accuracy=tensor(7.5246e-06)
DEBUG:root:i=500 accuracy=tensor(7.4413e-06)
DEBUG:root:i=501 accuracy=tensor(7.3615e-06)
DEBUG:root:i=502 accuracy=tensor(7.2818e-06)
DEBUG:root:i=503 accuracy=tensor(7.2022e-06)
DEBUG:root:i=504 accuracy=tensor(7.1250e-06)
DEBUG:root:i=505 accuracy=tensor(7.0453e-06)
DEBUG:root:i=506 accuracy=tensor(6.9696e-06)
DEBUG:root:i=507 accuracy=tensor(6.8945e-06)
DEBUG:root:i=508 accuracy=tensor(6.8187e-06)
DEBUG:root:i=509 accuracy=tensor(6.7448e-06)
DEBUG:root:i=510 accuracy=tensor(6.6715e-06)
DEBUG:root:i=511 accuracy=tensor(6.5988e-06)
DEBUG:root:i=512 accuracy=tensor(6.5272e-06)
DEBUG:root:i=513 accuracy=tensor(6.4574e-06)
DEBUG:root:i=514 accuracy=tensor(6.3864e-06)
DEBUG:root:i=515 accuracy=tensor(6.3180e-06)
DEBUG:root:i=516 accuracy=tensor(6.2490e-06)
DEBUG:root:i=517 accuracy=tensor(6.1811e-06)
DEBUG:root:i=518 accuracy=tensor(6.1130e-06)
DEBUG:root:i=519 accuracy=tensor(6.0470e-06)
DEBUG:root:i=520 accuracy=tensor(5.9819e-06)
DEBUG:root:i=521 accuracy=tensor(5.9180e-06)
DEBUG:root:i=522 accuracy=tensor(5.8536e-06)
DEBUG:root:i=523 accuracy=tensor(5.7894e-06)
DEBUG:root:i=524 accuracy=tensor(5.7266e-06)
DEBUG:root:i=525 accuracy=tensor(5.6639e-06)
DEBUG:root:i=526 accuracy=tensor(5.6031e-06)
DEBUG:root:i=527 accuracy=tensor(5.5426e-06)
DEBUG:root:i=528 accuracy=tensor(5.4837e-06)
DEBUG:root:i=529 accuracy=tensor(5.4226e-06)
DEBUG:root:i=530 accuracy=tensor(5.3639e-06)
DEBUG:root:i=531 accuracy=tensor(5.3057e-06)
DEBUG:root:i=532 accuracy=tensor(5.2467e-06)
DEBUG:root:i=533 accuracy=tensor(5.1899e-06)
DEBUG:root:i=534 accuracy=tensor(5.1346e-06)
DEBUG:root:i=535 accuracy=tensor(5.0785e-06)
DEBUG:root:i=536 accuracy=tensor(5.0224e-06)
DEBUG:root:i=537 accuracy=tensor(4.9695e-06)
DEBUG:root:i=538 accuracy=tensor(4.9156e-06)
DEBUG:root:i=539 accuracy=tensor(4.8616e-06)
DEBUG:root:i=540 accuracy=tensor(4.8089e-06)
DEBUG:root:i=541 accuracy=tensor(4.7568e-06)
DEBUG:root:i=542 accuracy=tensor(4.7049e-06)
DEBUG:root:i=543 accuracy=tensor(4.6534e-06)
DEBUG:root:i=544 accuracy=tensor(4.6030e-06)
DEBUG:root:i=545 accuracy=tensor(4.5546e-06)
DEBUG:root:i=546 accuracy=tensor(4.5036e-06)
DEBUG:root:i=547 accuracy=tensor(4.4559e-06)
DEBUG:root:i=548 accuracy=tensor(4.4085e-06)
DEBUG:root:i=549 accuracy=tensor(4.3605e-06)
DEBUG:root:i=550 accuracy=tensor(4.3118e-06)
DEBUG:root:i=551 accuracy=tensor(4.2651e-06)
DEBUG:root:i=552 accuracy=tensor(4.2190e-06)
DEBUG:root:i=553 accuracy=tensor(4.1735e-06)
DEBUG:root:i=554 accuracy=tensor(4.1288e-06)
DEBUG:root:i=555 accuracy=tensor(4.0841e-06)
DEBUG:root:i=556 accuracy=tensor(4.0387e-06)
DEBUG:root:i=557 accuracy=tensor(3.9942e-06)
DEBUG:root:i=558 accuracy=tensor(3.9541e-06)
DEBUG:root:i=559 accuracy=tensor(3.9088e-06)
DEBUG:root:i=560 accuracy=tensor(3.8668e-06)
DEBUG:root:i=561 accuracy=tensor(3.8246e-06)
DEBUG:root:i=562 accuracy=tensor(3.7835e-06)
DEBUG:root:i=563 accuracy=tensor(3.7408e-06)
DEBUG:root:i=564 accuracy=tensor(3.7018e-06)
DEBUG:root:i=565 accuracy=tensor(3.6601e-06)
DEBUG:root:i=566 accuracy=tensor(3.6241e-06)
DEBUG:root:i=567 accuracy=tensor(3.5828e-06)
DEBUG:root:i=568 accuracy=tensor(3.5441e-06)
DEBUG:root:i=569 accuracy=tensor(3.5053e-06)
DEBUG:root:i=570 accuracy=tensor(3.4673e-06)
DEBUG:root:i=571 accuracy=tensor(3.4297e-06)
DEBUG:root:i=572 accuracy=tensor(3.3931e-06)
DEBUG:root:i=573 accuracy=tensor(3.3562e-06)
DEBUG:root:i=574 accuracy=tensor(3.3207e-06)
DEBUG:root:i=575 accuracy=tensor(3.2832e-06)
DEBUG:root:i=576 accuracy=tensor(3.2488e-06)
DEBUG:root:i=577 accuracy=tensor(3.2131e-06)
DEBUG:root:i=578 accuracy=tensor(3.1773e-06)
DEBUG:root:i=579 accuracy=tensor(3.1440e-06)
DEBUG:root:i=580 accuracy=tensor(3.1092e-06)
DEBUG:root:i=581 accuracy=tensor(3.0759e-06)
DEBUG:root:i=582 accuracy=tensor(3.0457e-06)
DEBUG:root:i=583 accuracy=tensor(3.0112e-06)
DEBUG:root:i=584 accuracy=tensor(2.9769e-06)
DEBUG:root:i=585 accuracy=tensor(2.9435e-06)
DEBUG:root:i=586 accuracy=tensor(2.9126e-06)
DEBUG:root:i=587 accuracy=tensor(2.8817e-06)
DEBUG:root:i=588 accuracy=tensor(2.8519e-06)
DEBUG:root:i=589 accuracy=tensor(2.8203e-06)
DEBUG:root:i=590 accuracy=tensor(2.7885e-06)
DEBUG:root:i=591 accuracy=tensor(2.7592e-06)
DEBUG:root:i=592 accuracy=tensor(2.7347e-06)
DEBUG:root:i=593 accuracy=tensor(2.7002e-06)
DEBUG:root:i=594 accuracy=tensor(2.6707e-06)
DEBUG:root:i=595 accuracy=tensor(2.6398e-06)
DEBUG:root:i=596 accuracy=tensor(2.6145e-06)
DEBUG:root:i=597 accuracy=tensor(2.5832e-06)
DEBUG:root:i=598 accuracy=tensor(2.5605e-06)
DEBUG:root:i=599 accuracy=tensor(2.5302e-06)
DEBUG:root:i=600 accuracy=tensor(2.5007e-06)
DEBUG:root:i=601 accuracy=tensor(2.4753e-06)
DEBUG:root:i=602 accuracy=tensor(2.4478e-06)
DEBUG:root:i=603 accuracy=tensor(2.4209e-06)
DEBUG:root:i=604 accuracy=tensor(2.3952e-06)
DEBUG:root:i=605 accuracy=tensor(2.3686e-06)
DEBUG:root:i=606 accuracy=tensor(2.3486e-06)
DEBUG:root:i=607 accuracy=tensor(2.3194e-06)
DEBUG:root:i=608 accuracy=tensor(2.2948e-06)
DEBUG:root:i=609 accuracy=tensor(2.2705e-06)
DEBUG:root:i=610 accuracy=tensor(2.2516e-06)
DEBUG:root:i=611 accuracy=tensor(2.2234e-06)
DEBUG:root:i=612 accuracy=tensor(2.1962e-06)
DEBUG:root:i=613 accuracy=tensor(2.1714e-06)
DEBUG:root:i=614 accuracy=tensor(2.1479e-06)
DEBUG:root:i=615 accuracy=tensor(2.1256e-06)
DEBUG:root:i=616 accuracy=tensor(2.1025e-06)
DEBUG:root:i=617 accuracy=tensor(2.0785e-06)
DEBUG:root:i=618 accuracy=tensor(2.0567e-06)
DEBUG:root:i=619 accuracy=tensor(2.0350e-06)
DEBUG:root:i=620 accuracy=tensor(2.0121e-06)
DEBUG:root:i=621 accuracy=tensor(1.9924e-06)
DEBUG:root:i=622 accuracy=tensor(1.9697e-06)
DEBUG:root:i=623 accuracy=tensor(1.9463e-06)
DEBUG:root:i=624 accuracy=tensor(1.9273e-06)
DEBUG:root:i=625 accuracy=tensor(1.9057e-06)
DEBUG:root:i=626 accuracy=tensor(1.8847e-06)
DEBUG:root:i=627 accuracy=tensor(1.8674e-06)
DEBUG:root:i=628 accuracy=tensor(1.8439e-06)
DEBUG:root:i=629 accuracy=tensor(1.8254e-06)
DEBUG:root:i=630 accuracy=tensor(1.8045e-06)
DEBUG:root:i=631 accuracy=tensor(1.7848e-06)
DEBUG:root:i=632 accuracy=tensor(1.7669e-06)
DEBUG:root:i=633 accuracy=tensor(1.7462e-06)
DEBUG:root:i=634 accuracy=tensor(1.7290e-06)
DEBUG:root:i=635 accuracy=tensor(1.7128e-06)
DEBUG:root:i=636 accuracy=tensor(1.6912e-06)
DEBUG:root:i=637 accuracy=tensor(1.6721e-06)
DEBUG:root:i=638 accuracy=tensor(1.6540e-06)
DEBUG:root:i=639 accuracy=tensor(1.6386e-06)
DEBUG:root:i=640 accuracy=tensor(1.6174e-06)
DEBUG:root:i=641 accuracy=tensor(1.6007e-06)
DEBUG:root:i=642 accuracy=tensor(1.5874e-06)
DEBUG:root:i=643 accuracy=tensor(1.5687e-06)
DEBUG:root:i=644 accuracy=tensor(1.5501e-06)
DEBUG:root:i=645 accuracy=tensor(1.5328e-06)
DEBUG:root:i=646 accuracy=tensor(1.5174e-06)
DEBUG:root:i=647 accuracy=tensor(1.5021e-06)
DEBUG:root:i=648 accuracy=tensor(1.4824e-06)
DEBUG:root:i=649 accuracy=tensor(1.4676e-06)
DEBUG:root:i=650 accuracy=tensor(1.4509e-06)
DEBUG:root:i=651 accuracy=tensor(1.4373e-06)
DEBUG:root:i=652 accuracy=tensor(1.4207e-06)
DEBUG:root:i=653 accuracy=tensor(1.4067e-06)
DEBUG:root:i=654 accuracy=tensor(1.3897e-06)
DEBUG:root:i=655 accuracy=tensor(1.3756e-06)
DEBUG:root:i=656 accuracy=tensor(1.3622e-06)
DEBUG:root:i=657 accuracy=tensor(1.3446e-06)
DEBUG:root:i=658 accuracy=tensor(1.3308e-06)
DEBUG:root:i=659 accuracy=tensor(1.3166e-06)
DEBUG:root:i=660 accuracy=tensor(1.3049e-06)
DEBUG:root:i=661 accuracy=tensor(1.2885e-06)
DEBUG:root:i=662 accuracy=tensor(1.2741e-06)
DEBUG:root:i=663 accuracy=tensor(1.2606e-06)
DEBUG:root:i=664 accuracy=tensor(1.2485e-06)
DEBUG:root:i=665 accuracy=tensor(1.2336e-06)
DEBUG:root:i=666 accuracy=tensor(1.2197e-06)
DEBUG:root:i=667 accuracy=tensor(1.2067e-06)
DEBUG:root:i=668 accuracy=tensor(1.1934e-06)
DEBUG:root:i=669 accuracy=tensor(1.1815e-06)
DEBUG:root:i=670 accuracy=tensor(1.1673e-06)
DEBUG:root:i=671 accuracy=tensor(1.1580e-06)
DEBUG:root:i=672 accuracy=tensor(1.1443e-06)
DEBUG:root:i=673 accuracy=tensor(1.1315e-06)
DEBUG:root:i=674 accuracy=tensor(1.1186e-06)
DEBUG:root:i=675 accuracy=tensor(1.1070e-06)
DEBUG:root:i=676 accuracy=tensor(1.0924e-06)
DEBUG:root:i=677 accuracy=tensor(1.0964e-06)
DEBUG:root:i=678 accuracy=tensor(1.0743e-06)
DEBUG:root:i=679 accuracy=tensor(1.0608e-06)
DEBUG:root:i=680 accuracy=tensor(1.0498e-06)
DEBUG:root:i=681 accuracy=tensor(1.0388e-06)
DEBUG:root:i=682 accuracy=tensor(1.0269e-06)
DEBUG:root:i=683 accuracy=tensor(1.0217e-06)
DEBUG:root:i=684 accuracy=tensor(1.0036e-06)
DEBUG:root:i=685 accuracy=tensor(9.9388e-07)
INFO:root:rank=0 pagerank=0.7014884352684021 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=0.7014861702919006 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=0.1055154949426651 url=www.lawfareblog.com/cost-using-zero-days
INFO:root:rank=3 pagerank=0.03175726532936096 url=www.lawfareblog.com/lawfare-podcast-former-congressman-brian-baird-and-daniel-schuman-how-congress-can-continue-function
INFO:root:rank=4 pagerank=0.022039875388145447 url=www.lawfareblog.com/events
INFO:root:rank=5 pagerank=0.016026968136429787 url=www.lawfareblog.com/water-wars-increased-us-focus-indo-pacific
INFO:root:rank=6 pagerank=0.016026228666305542 url=www.lawfareblog.com/water-wars-drill-maybe-drill
INFO:root:rank=7 pagerank=0.0160230603069067 url=www.lawfareblog.com/water-wars-disjointed-operations-south-china-sea
INFO:root:rank=8 pagerank=0.01602022908627987 url=www.lawfareblog.com/water-wars-song-oil-and-fire
INFO:root:rank=9 pagerank=0.016020221635699272 url=www.lawfareblog.com/water-wars-sinking-feeling-philippine-china-relations
```
  
   Task 2, part 1:
```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona'
   
   INFO:root:rank=0 pagerank=0.6233850717544556 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=0.6233609914779663 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=0.13966940343379974 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 pagerank=0.10857659578323364 url=www.lawfareblog.com/rational-security-my-corona-edition
INFO:root:rank=4 pagerank=0.10857659578323364 url=www.lawfareblog.com/brexit-not-immune-coronavirus
INFO:root:rank=5 pagerank=0.08247543126344681 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=6 pagerank=0.07788563519716263 url=www.lawfareblog.com/britains-coronavirus-response
INFO:root:rank=7 pagerank=0.07788563519716263 url=www.lawfareblog.com/prosecuting-purposeful-coronavirus-exposure-terrorism
INFO:root:rank=8 pagerank=0.07549146562814713 url=www.lawfareblog.com/trump-asks-supreme-court-stay-congressional-subpeona-tax-returns
INFO:root:rank=9 pagerank=0.07490721344947815 url=www.lawfareblog.com/house-oversight-committee-holds-day-two-hearing-government-coronavirus-response
```
  

   Task 2, part 2:
  ```
   $ python3 pagerank.py --data=data/lawfareblog.csv.gz --filter_ratio=0.2 --personalization_vector_query='corona' --search_query='-corona'
   INFO:root:rank=0 pagerank=6.3127e-01 url=www.lawfareblog.com/covid-19-speech-and-surveillance-response
INFO:root:rank=1 pagerank=6.3124e-01 url=www.lawfareblog.com/lawfare-live-covid-19-speech-and-surveillance
INFO:root:rank=2 pagerank=1.5947e-01 url=www.lawfareblog.com/chinatalk-how-party-takes-its-propaganda-global
INFO:root:rank=3 pagerank=9.3360e-02 url=www.lawfareblog.com/trump-cant-reopen-country-over-state-objections
INFO:root:rank=4 pagerank=7.0277e-02 url=www.lawfareblog.com/fault-lines-foreign-policy-quarantined
INFO:root:rank=5 pagerank=6.9713e-02 url=www.lawfareblog.com/lawfare-podcast-mom-and-dad-talk-clinical-trials-pandemic
INFO:root:rank=6 pagerank=6.4944e-02 url=www.lawfareblog.com/limits-world-health-organization
INFO:root:rank=7 pagerank=5.9492e-02 url=www.lawfareblog.com/chinatalk-dispatches-shanghai-beijing-and-hong-kong
INFO:root:rank=8 pagerank=5.1245e-02 url=www.lawfareblog.com/us-moves-dismiss-case-against-company-linked-ira-troll-farm
INFO:root:rank=9 pagerank=5.1245e-02 url=www.lawfareblog.com/livestream-house-armed-services-committee-holds-hearing-priorities-missile-defense
  ```
   

1. Ensure that all your changes to the `pagerank.py` and `README.md` files are committed to your repo and pushed to github.

1. Get at least 5 stars on your repo.
   (You made trade stars with other students in the class.)

   > **NOTE:**
   > 
   > Recruiters use github profiles to determine who to hire,
   > and pagerank is used to rank user profiles and projects.
   > Links in this graph correspond to who has starred/followed who's repo.
   > By getting more stars on your repo, you'll be increasing your github pagerank, which increases the likelihood that recruiters will hire you.
   > To see an example, [perform a search for `data mining`](https://github.com/search?q=data+mining).
   > Notice that the results are returned "approximately" ranked by the number of stars,
   > but because "some stars count more than others" the results are not exactly ranked by the number of stars.
   > (I asked you not to fork this repo because forks are ranked lower than non-forks.)
   >
   > In some sense, we are doing a "dual problem" to data mining by getting these stars.
   > Recruiters are using data mining to find out who the best people to recruit are,
   > and we are hacking their data mining algorithms by making those algorithms select you instead of someone else.
   >
   > If you're interested in exploring this idea further, here's a python tutorial for extracting GitHub's social graph: <https://www.oreilly.com/library/view/mining-the-social/9781449368180/ch07.html> ; if you're interested in learning more about how recruiters use github profiles, read this Hacker News post: <https://news.ycombinator.com/item?id=19413348>.

1. Submit the url of your repo to sakai.

   Each part is worth 2 points, for 12 points overall.
