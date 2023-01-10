---
layout: post
title: "Code Review for Research Collaboration"
subtitle: "Code Review in R to Ensure Reproducibility and Accuracy of Statistical Analyses in Research"
date: 2023-01-09 14:45:13 -0400
background: '/img/posts/pexels-sora-shimazaki-5926393.jpg'
---

<p>This post is a summary of the code review / cross-validation process of a monumental reserach project that involves a 140+ researchers from different fields of social science and thousands of lines to review. The project is published in a prestigous journal, and here is the <a href="https://psyarxiv.com/wdxsb/">preprint of the project</a>.<p>

<h2 class="section-heading">Why Code Review</h2>

<p>Reproducibility: Code review can help ensure that the code is written in a clear and well-documented manner, which makes it easier for others to understand and reproduce the results.<p>

<p>Accuracy: Code review can help identify any mistakes or errors in the code, which can affect the accuracy of the results. By having another researcher review the code, it is more likely that any mistakes will be caught and corrected.<p>

<p>Best practices: Code review can also help promote the use of best practices in coding, such as using appropriate data structures and algorithms, and following a consistent coding style.<p>


<img class="img-fluid" src="https://github.com/EvieXinqiGuo/eviexinqiguo.github.io/blob/master/img/posts/code-review.png?raw=true" alt="Demo Image">

<img src="/workspaces/eviexinqiguo.github.io/_site/img/posts/code-review.png" alt="">
<span class="caption text-muted">The emotional burden of potentially exposing buggy, poorly-written code to peers can be a barrier for researchers to adopt code review.</span>

<h2 class="section-heading">My Role in the Code Review Process</h2>

<p>An extensive code review was done during the preparation of the manuscript to assess the reproducibility of the code and cross-validation. Two tournament participants volunteered for the code review after the main authors reached out. The two code reviewers downloaded and ran the data (3 data files in .csv format) and code (1 Rmarkdown file) shared through the Github repository by the main authors. The code reviewers carried out the code check on their own computers (i.e., in different environments from where the code was initially developed), and let the main authors know that the code was possible to run, and the code did carry out the intended analyses.</p>

</p>The shared code was divided into sections that correspond to the statistical output and visualization in the manuscript. If the code reviewers spotted an error, they were instructed to take a note of the line and then write a short description that explains what the potential issues are. Minor coding errors that did not impact the manuscript results and were found and reported to the main authors. The short description of the errors was emailed to the main authors and the code updates were pushed through Github. When needed, the code reviewers also made efforts to improve the readability of the code by breaking up long lines of code and adding comments.</p>

<blockquote class="blockquote">I advocate for more internal and external code-review. As I believe that scrutinizing and improving the code that drives analyses elevates the quality and reliability of research to new heights.</blockquote>


</p>Image by the author</p>