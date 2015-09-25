---
layout: post
title: "Smashing BigData! – Our Concurrent Scraper written in Scala/Akka Framework"
description: "Smashing BigData! – Our Concurrent scraper written in Scala/Akka Framework for processing of Common Crawl Web Data"
category: [NLP]
tags: [NLP, Text Mining, Web Crawler, Scala, Akka Framework]
---

###Motivation
We are given a task – to construct N-grams for Polish language based on Polish Internet Websites. One could say: “It is hardly possible to be done”, because first you have to scrape all the Internet content consisting of nearly petabytes of data, such storage would be very demanding and challenging i.e. expensive infrastructure by the means of hardware and software, a need of well-trained professionals for managing etc. 
Fortunately, somebody has already done this tremendous task! [The Common Crawl Foundation](http://commoncrawl.org/) comes forward with its Web Data Crawl. 

<!--more--> 

###BigData Source
Now, a few words about Common Crawl Project. It is a non-profit foundation registered in California for producing and maintaining open repository of web crawl data which is widely accessible and analyzable. They build up very useful resource for innovative research, and for applications in business and education.

Web Data Crawl has being collected over 7 years, the project has evolved over the years, with some changes in data formats and metadata used. The Data is divided into segments what comes naturally, since fairly two years the data dump is done once a month. Each segment can extent to hundreds terabytes of data (including all formats). The Common Crawl Foundation provides a few formats for their data. We have chosen a WET format, as our task only requires textual information. WET files contain only plaintext without any markup tags. At the top, there is meta information which is followed by website content. More details are available [here](http://commoncrawl.org/the-data/get-started).

As we would like to process the Common Crawl Web Data, we have to get this crawl at first! Since we would like to skip using Amazon S3, we have to download the data by ourselves (and that's why we need a scraper).  

###Scraper Architecture 
Scraper architecture consists of several components:

* Main Server as a scraper habitat
* Cassandra Storage
* Common Crawl Storage Servers

####Main Server

Main Server &minus; it's a core of the system, on which scraper performs its tasks and language detection takes place. It is an abstract term since it is built up from a few computers stored in a rack and makes the environment for scraper existence. Currently, the computational resources available within the main server are: Intel(R) Xeon(R) CPU E5-2420 v2 @ 2.20GHz 16 cores and 384 GB RAM. However, the main server is expected to be enlarged significantly in the near future.

Since the enormous data volume, scraping activities take place every day after working hours in the Institute ![smile]({{site.url }}/assets/images/smile.png) (After scraping actors are launched, the Internet connection quality is much reduced.) Crontab scripts take care about both firing scraping up and turning it down.

Highly concurrent processing of Web Data Crawl goes on. As we have used actor based Akka Framework we got it out of the box.

The main server is a habitat for Actors responsible for downloading and processing data, and for Polish language detection.

####Actors which are involved in processing:

* *ActorSystem* which plays the role of the application's CEO ![wink]({{site.url }}/assets/images/wink.png) 

It is the core of the scraping system which starts up the scraper, and release resources at the end, by scraping system termination. Data base connection is established and *FileMaster* actor is created. After all, the *StartDownloading* message is sent to the *FileMaster* for further actions.

* *FileMaster* actor is a kind of application's manager, who supervises the tasks of his workers. We can call him a master and its workers as slaves, with accordance to parallel computing nomenclature. 

This is the real manager who creates its workers, hires them, and gives them tasks to be done. (As it is very important function, let's call him as *Master*.) After *Master* has  received the message *StartDownloading* from the *ActorSystem*, he creates a proper number of his faithful slaves *FileWorkers*, gets the URLs (from wet.paths, file containing URLs for parts of data crawl segment) for feeding them by sending *ProcessFile* message, and in the end begins processing, which is about iterating over available *FileWorkers* and giving them a new task (URL). In the meanwhile, *FileMaster* is listening whether *ProcessingFinished* message occurred, if so *Master* deletes the downloaded file, pushes the info to database that a specific URL has been processed, and checks whether or not all processing is already done.

* *FileWorker* is a worker responsible for data processing. This actor is called as *Slave* and is under supervision of his *Master*. There might be a lot of them. In our system we have 48 *Slaves*, what is equal to 3 times the number of CPU cores.

*Slave* performs most of the job, but not all... After he has received *ProcessFile* message from his *Master*, the processing step is initialized. First, the file containing gzipped data is downloaded, and second he iterates over unzipped file stream containing Web Data Crawl, which is about 300 MB raw text, and fetches chunks of plaintext, with special care of headers, and send further to *Bouncer* actors for Polish language detection. Each chunk of text is sent to currently available *Bouncer* actor. After obtaining a response from *Bouncer*, *FileWorker* sends the message to his *Master* with a proper feedback (Failure or Success).


![Scraper Architecture]({{ site.url }}/assets/images/scraper-architecture.png)
