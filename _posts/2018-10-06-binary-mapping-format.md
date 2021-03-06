---
layout: post
title: PAF I save 90 % of disc space
date: 2018-06-18
draft: true
published: true
tags: draft overlapper long-read compression
---

{% include setup %}

# PAF I save 90 % of disc space

During last week Shaun Jack post this tweet:

<blockquote class="twitter-tweet" data-lang="fr">
<p lang="en" dir="ltr">
I have a 1.2 TB PAF.gz file of minimap2 all-vs-all alignments of 18 flowcells of Oxford Nanopore reads. Yipes. I believe that&#39;s my first file to exceed a terabyte. Is there a better way? Perhaps removing the subsumed reads before writing the all-vs-all alignments to disk?</p>&mdash; Shaun Jackman (@sjackman) <a href="https://twitter.com/sjackman/status/1047729989318139904?ref_src=twsrc%5Etfw">4 octobre 2018</a>
</blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

For people who didn't work on long-read assembly, [Minimap](https://github.com/lh3/minimap2) is a very good mapper used to find similar region between long read. It's output are PAF files (Pairwise Alignment Format) and summarize on [minimap2 man page](https://lh3.github.io/minimap2/minimap2.html#10). Roughly it's tsv file which store for each similar region found (called before match): two reads names, reads length, begin and end position of match, plus some other information.  

This tweet creates some discussion and third solution was proposed like using classic mapping against reference compression format, filter some match, the creation of a new binary compressed format to store all-vs-all mapping.

## Use mapping reference file

Torsten Seemann and I suggest using SAM minimap output and compress it in BAM/CRAM format. But apparently it isn't working, well.

<blockquote class="twitter-tweet" data-lang="fr">
<p lang="en" dir="ltr">My trouble to convert bam in cram is solved thank to <a href="https://twitter.com/RayanChikhi?ref_src=twsrc%5Etfw">@RayanChikhi</a> !<br><br>minimap output without sequence compress in cram,noref (i.e. tmp_short.cram) is little bit heavier than classic paf compress in gz.<br><br>So it&#39;s probably time for a Binary Alignment Format. <a href="https://t.co/W02wj7tf2I">pic.twitter.com/W02wj7tf2I</a></p>&mdash; Pierre Marijon (@pierre_marijon) <a href="https://twitter.com/pierre_marijon/status/1047798695822024704?ref_src=twsrc%5Etfw">4 octobre 2018</a>
</blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

OK I have not removed necessary field in SAM format (sequence and mapping field), and with better compression solution, It isn't better than PAF format compression with gzip. Maybe with larger file this solution could save some space.

## Filter

Heng Li say :

<blockquote class="twitter-tweet" data-lang="fr">
<p lang="en" dir="ltr">Minimap2 reports hits, not only overlaps. The great majority of hits are non-informative to asm. Hits involving repeats particularly hurt. For asm, there are ways to reduce them, but that needs evaluation. I won&#39;t go into that route soon because ... 1/2</p>&mdash; Heng Li (@lh3lh3) <a href="https://twitter.com/lh3lh3/status/1047823011527753728?ref_src=twsrc%5Etfw">4 octobre 2018</a>
</blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Minimap didn't make any assumption about what user wants to do with read matching, and it's a very good thing but sometimes you store too much information for your usage. So filtering overlap before storing it on your disk, could by a good idea.

Awk, Bash, Python, {choose your language} script could do this job perfectly.

Alex Di Genov [suggest using minimap2 API](https://twitter.com/digenoma/status/1047852263111385088) to build a special minimap with integrating filters, this solution has probably better performance than *ad hoc* script but it's less flexible you need use minimap2.

My solution is a little soft in rust [fpa (Filter Pairwise Alignment)](https://github.com/natir/fpa), fpa take as input the paf or mhap, and this can filter match by:
- type: containment, internal-match, dovetail
- length: match is upper or lower than a threshold
- read name: match against a regex, it's a read match against him

fpa is available in bioconda and in cargo.

OK filter match is easy and we have many available solution. 

## A Binary Alignment Fromat

<blockquote class="twitter-tweet" data-lang="fr">
<p lang="en" dir="ltr">Is it time for a Binary Alignment Format that uses integer compression techniques?</p>&mdash; Rayan Chikhi (@RayanChikhi) <a href="https://twitter.com/RayanChikhi/status/1047773219086897153?ref_src=twsrc%5Etfw">4 octobre 2018</a>
</blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

This is the question they initiate this blog post.

Her I just want to present a little investigation, about how we can compress Pairwise Alignment, I call this format jPAF and it's just a POC and this never changes.

Rougly jPAF is a json compressed format, so it isn't really a binary, but I just wanted to test some ideas... So it's cool.

We have 3 main object in this json:
- header\_index: a dict they associate an index to a header name
- reads\_index: associate a read name and her length to an index
- matchs: a list of match, a match header\_index

A little example are probably better:

```
{
   "header_index":{
      "0":"read_a",
      "1":"begin_a",
      "2":"end_a",
      "3":"strand",
      "4":"read_b",
      "5":"begin_b",
      "6":"end_b",
      "7":"nb_match",
      "8":"match_length",
      "9":"qual"
   },
   "read_index":[
      {
         "name":"1_2",
         "len":5891
      },
      {
         "name":"2_3",
         "len":4490
      },
   ],
   "match":[
      [
         0,
         1642,
         5889,
         "+",
         1,
         1,
         4248,
         4247,
         4247,
         0,
         "tp:A:S"
      ],
   ]
}
```

Her we have a PAF like header, two read 1_2 and 2_3 with 5891 and 4490 base respectively and one overlap with length 4247 base in the same strand between them.

jPAF are fully inspired by PAF same field name I just take the PAF convert it in json and add two little trick to save space.

First tricks write read names and read length one time.
Second tricks is more json trick, first time each record are a dictionary with keyname associate to value, with this solution baf is heaviest than paf. If I associate a field name to a index, I can store records in table not in json object and avoid redundancy.

OK I have a pretty cool format, to avoid some repetition without loss of information, but I really save space or not ?

## Result

Dataset: For this I reuse the same dataset as may previous blog post. It's two real dataset pacbio one and nanopore one.

Mapping : I run minimap2 mapping with preset ava-pb and ava-ont for pacbio and nanopore respectively

This table presents file size and space savings against some other file. SAM bam and cram files are long or short, in long we keep all data present in minimap2 output, in short we replace sequence and quality by a \*.

If you want to replicate results just follow instruction avaible at [github repro](https://github.com/natir/jPAF).

## Discussion

Minimap generate to much match, but it's easy to remove unusual match, with fpa or with *ad hoc* tools. The format design to store mapping against reference isn't better than actual format compress with generalist algorithm.

The main problem of jPaf is many quality, read\_index it's save disk space but loss in time and RAM. You can't stream out this format, you need to wait until all alignment are found before writing, when you need to read file you need to keep read\_index in RAM any time. 

But I written it in two hours, the format is lossless and they save **33 to 64 %** of disk space, depend of compression used. 

Actually I'm not sure we need binary compressed format for store pairwise alignment against read, but in future with more and more long-read data we probably need it. This result encourages me to continue on this approch and read more thing around bam/cram canu ovlStore.

If you want search with me and discussion about Pairwise Aligment format comment section is available.
