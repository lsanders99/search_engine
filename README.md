# HW: Search Engine

In this assignment you will create a highly scalable web search engine.

**Due Date:** Sunday, 9 May

**Learning Objectives:**
1. Learn to work with a moderate large software project
1. Learn to parallelize data analysis work off the database
1. Learn to work with WARC files and the multi-petabyte common crawl dataset
1. Increase familiarity with indexes and rollup tables for speeding up queries

## Task 0: project setup

1. Fork this github repo, and clone your fork onto the lambda server

1. Ensure that you'll have enough free disk space by:
    1. bring down any running docker containers
    1. run the command
       ```
       $ docker system prune
       ```

## Task 1: getting the system running

In this first task, you will bring up all the docker containers and verify that everything works.

There are three docker-compose files in this repo:
1. `docker-compose.yml` defines the database and pg_bouncer services
1. `docker-compose.override.yml` defines the development flask web app
1. `docker-compose.prod.yml` defines the production flask web app served by nginx

Your tasks are to:

1. Modify the `docker-compose.override.yml` file so that the port exposed by the flask service is different.

1. Run the script `scripts/create_passwords.sh` to generate a new production password for the database.

1. Build and bring up the docker containers.

1. Enable ssh port forwarding so that your local computer can connect to the running flask app.

1. Use firefox on your local computer to connect to the running flask webpage.
   If you've done the previous steps correctly,
   all the buttons on the webpage should work without giving you any error messages,
   but there won't be any data displayed when you search.

1. Run the script
   ```
   $ sh scripts/check_web_endpoints.sh
   ```
   to perform automated checks that the system is running correctly.
   All tests should report `[pass]`.

## Task 2: loading data

There are two services for loading data:
1. `downloader_warc` loads an entire WARC file into the database; typically, this will be about 100,000 urls from many different hosts. 
1. `downloader_host` searches the all WARC entries in either the common crawl or internet archive that match a particular pattern, and adds all of them into the database

### Task 2a

We'll start with the `downloader_warc` service.
There are two important files in this service:
1. `services/downloader_warc/downloader_warc.py` contains the python code that actually does the insertion
1. `downloader_warc.sh` is a bash script that starts up a new docker container connected to the database, then runs the `downloader_warc.py` file inside that container

Next follow these steps:
1. Visit https://commoncrawl.org/the-data/get-started/
1. Find the url of a WARC file.
   On the common crawl website, the paths to WARC files are referenced from the Amazon S3 bucket.
   In order to get a valid HTTP url, you'll need to prepend `https://commoncrawl.s3.amazonaws.com/` to the front of the path.
1. Then, run the command
   ```
   $ ./download_warc.sh $URL
   ```
   where `$URL` is the url to your selected WARC file.
1. Run the command
   ```
   $ docker ps
   ```
   to verify that the docker container is running.
1. Repeat these steps to download at least 5 different WARC files, each from different years.
   Each of these downloads will spawn its own docker container and can happen in parallel.

You can verify that your system is working with the following tasks.
(Note that they are listed in order of how soon you will start seeing results for them.)
1. Running `docker logs` on your `download_warc` containers.
1. Run the query
   ```
   SELECT count(*) FROM metahtml;
   ```
   in psql.
1. Visit your webpage in firefox and verify that search terms are now getting returned.

### Task 2b

The `download_warc` service above downloads many urls quickly, but they are mostly low-quality urls.
For example, most URLs do not include the date they were published, and so their contents will not be reflected in the ngrams graph.
In this task, you will implement and run the `download_host` service for downloading high quality urls.

1. The file `services/downloader_host/downloader_host.py` has 3 `FIXME` statements.
   You will have to complete the code in these statements to make the python script correctly insert WARC records into the database.

   HINT:
   The code will require that you use functions from the cdx_toolkit library.
   You can find the documentation [here](https://pypi.org/project/cdx-toolkit/).
   You can also reference the `download_warc` service for hints,
   since this service accomplishes a similar task.

1. Run the query
   ```
   SELECT * FROM metahtml_test_summary_host;
   ```
   to display all of the hosts for which the metahtml library has test cases proving it is able to extract publication dates.
   Note that the command above lists the hosts in key syntax form, and you'll have to convert the host into standard form.
1. Select 5 hostnames from the list above, then run the command
   ```
   $ ./downloader_host.sh "$HOST/*"
   ```
   to insert the urls from these 5 hostnames.

## ~~Task 3: speeding up the webpage~~

Since everyone seems pretty overworked right now,
I've done this step for you.

There are two steps:
1. create indexes for the fast text search
1. create materialized views for the `count(*)` queries

## Submission

1. Edit this README file with the results of the following queries in psql.
   The results of these queries will be used to determine if you've completed the previous steps correctly.

    1. This query shows the total number of webpages loaded:
       ```
       select count(*) from metahtml;
       ```

       ```
        count
       --------
        226774
       (1 row)
       ```

    1. This query shows the number of webpages loaded / hour:
       ```
       select * from metahtml_rollup_insert order by insert_hour desc limit 100;
       ```

       ```
        hll_count |  url  | hostpathquery | hostpath | host  |      insert_hour       
       -----------+-------+---------------+----------+-------+------------------------
                5 |  2382 |          2405 |     2071 |     5 | 2021-05-03 16:00:00+00
                5 | 29066 |         28568 |    21744 |     5 | 2021-05-03 15:00:00+00
                5 | 29061 |         30155 |    22948 |     5 | 2021-05-03 14:00:00+00
                5 | 30016 |         28893 |    22722 |     5 | 2021-05-03 13:00:00+00
                5 | 30478 |         29822 |    23129 |     5 | 2021-05-03 12:00:00+00
                5 |  6910 |          7075 |     6429 |     5 | 2021-05-03 11:00:00+00
                1 | 25938 |         26327 |    25676 | 22399 | 2021-05-02 22:00:00+00
                3 | 56530 |         59380 |    50985 | 32086 | 2021-05-02 21:00:00+00
       (8 rows)
       ```

    1. This query shows the hostnames that you have downloaded the most webpages from:
       ```
       select * from metahtml_rollup_host order by hostpath desc limit 100;
       ```

       ```
         url  | hostpathquery | hostpath |            host            
       -------+---------------+----------+----------------------------
        27239 |         25569 |    25572 | uk,co,bbc)
        21882 |         22016 |    22016 | com,worldatlas)
        19090 |         18721 |    18721 | com,21stcenturywire)
         4754 |          4837 |     4827 | net,earthreview)
        29274 |         29062 |      918 | au,com,sbs)
           47 |            47 |       40 | com,atgstores)
           32 |            32 |       32 | com,agoda)
           36 |            36 |       30 | com,mlb)
           30 |            30 |       30 | com,scribd)
           29 |            29 |       29 | org,wikipedia,en)
           30 |            30 |       29 | com,go,espn)
           29 |            29 |       29 | com,theguardian)
           32 |            32 |       29 | com,cengage,community)
           27 |            27 |       27 | org,wikipedia,de)
           27 |            27 |       27 | com,dollartree)
           27 |            27 |       27 | id,co,tripadvisor)
           27 |            27 |       27 | org,worldcat)
           25 |            25 |       25 | com,landsend)
           26 |            26 |       25 | com,google)
           24 |            24 |       24 | com,thefind)
           24 |            24 |       24 | com,pandora)
           24 |            24 |       24 | com,newsok)
           24 |            24 |       24 | com,reddit)
           23 |            23 |       23 | com,packersproshop)
           23 |            23 |       23 | com,fanatics)
           23 |            23 |       23 | com,6pm)
           23 |            23 |       23 | kr,co,tripadvisor)
           22 |            22 |       22 | es,tripadvisor)
           22 |            22 |       22 | com,weather)
           22 |            22 |       22 | com,sports-reference)
           22 |            22 |       22 | com,gilt)
           21 |            21 |       21 | com,beeradvocate)
           21 |            21 |       21 | com,streetsideauto)
           22 |            22 |       21 | com,weatherbug,weather)
           21 |            21 |       21 | com,tampabay)
           22 |            22 |       21 | com,dpreview)
           21 |            21 |       21 | com,stackoverflow)
           20 |            20 |       20 | com,bloomberg)
           20 |            20 |       20 | com,imdb)
           20 |            20 |       20 | com,oxforddictionaries)
           20 |            20 |       20 | gov,clinicaltrials)
           20 |            20 |       20 | com,northerntool)
           26 |            26 |       20 | com,cnet)
           20 |            20 |       20 | com,wsj,online)
           19 |            19 |       19 | br,com,tripadvisor)
           19 |            19 |       19 | com,vimeo)
           19 |            19 |       19 | mx,com,tripadvisor)
           19 |            19 |       19 | com,utsandiego)
           19 |            19 |       19 | com,boston)
           19 |            19 |       19 | dk,tripadvisor)
           19 |            19 |       19 | com,oracle,docs)
           19 |            19 |       19 | com,dreamstime)
           19 |            19 |       19 | com,tv)
           23 |            23 |       19 | com,whitepages)
           18 |            18 |       18 | com,moultrienews)
           18 |            18 |       18 | org,faqs)
           18 |            18 |       18 | org,metoperashop)
           18 |            18 |       18 | ve,com,tripadvisor)
           19 |            19 |       18 | org,marylandpublicschools)
           18 |            18 |       18 | org,eol)
           18 |            18 |       18 | com,businessinsider)
           18 |            18 |       18 | com,ign)
           18 |            18 |       18 | com,scrapbook,store)
           18 |            18 |       18 | com,123rf)
           18 |            18 |       18 | com,appszoom)
           18 |            18 |       18 | com,tripadvisor,pl)
           18 |            18 |       18 | com,wsj,blogs)
           18 |            18 |       18 | com,modelmayhem)
           18 |            18 |       18 | uk,co,theregister)
           17 |            17 |       17 | it,tripadvisor)
           17 |            17 |       17 | ar,com,tripadvisor)
           17 |            17 |       17 | edu,cornell,law)
           27 |            27 |       17 | com,nytimes)
           17 |            17 |       17 | nl,tripadvisor)
           17 |            17 |       17 | com,champssports)
           17 |            17 |       17 | com,the-house)
           22 |            22 |       17 | com,yardbarker)
           17 |            17 |       17 | com,bleacherreport)
           17 |            17 |       17 | com,myfitnesspal)
           20 |            20 |       17 | com,musicnotes)
           17 |            17 |       17 | com,britannica)
           17 |            17 |       17 | org,wikidata)
           17 |            17 |       17 | com,loft)
           16 |            16 |       16 | se,tripadvisor)
           16 |            16 |       16 | net,styleforum)
           16 |            16 |       16 | org,wikipedia,fr)
           16 |            16 |       16 | org,dmoz)
           16 |            16 |       16 | com,mobygames)
           16 |            16 |       16 | com,gamezone)
           21 |            21 |       16 | com,mocpages)
           17 |            17 |       16 | edu,umich,icpsr)
           16 |            16 |       16 | com,rep-am)
           16 |            16 |       16 | com,defensereview)
           18 |            18 |       16 | com,snagajob)
           16 |            16 |       16 | net,airliners)
           16 |            16 |       16 | net,mangareader)
           18 |            18 |       16 | com,godlikeproductions)
           16 |            16 |       16 | com,neimanmarcus)
           20 |            20 |       16 | fr,insee,recherche-naf)
           16 |            16 |       16 | com,gawker)
       (100 rows)
       ```

1. Take a screenshot of an interesting search result.
   Add the screenshot to your git repo, and modify the `<img>` tag below to point to the screenshot.

   <img src='screenshot.png' />

1. Commit and push your changes to github.

1. Submit the link to your github repo in sakai.
