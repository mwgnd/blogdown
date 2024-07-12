---
title: Scrape Telegram messages via reticulate in R
author: Manuel
date: '2024-05-24'
slug: []
categories:
  - R
tags:
  - r
  - telegram
  - reticulate
---

First post on this blogdown site. Hello World!

As I write this, I have just finished my studies. In my master's thesis, I worked heavily with Telegram data because. As a social scientist interested in right-wing populist conspiracy discourse, I wanted to analyze the communication of far-right movements on Telegram. Although I am used to working with R for text analysis, I had to use Python for scraping the messages. For this purpose, I used the [Telethon](https://docs.telethon.dev/en/stable/) library for interacting with the Telegram API. But because I'm more into R, and wanted to use that in the first place, I recently tried scraping with the `reticulate` package in R. In this post, I will show you how to scrape Telegram messages using Telethon via `reticulate` in R. For easy switching between R and Python, I use R Markdown, where i also wrote this post.

# Setup reticulate

You need Python installed on your system to use `reticulate`. After installing `reticulate`, you can create a Python virtual environment with the `virtualenv_create()` function. For using Telethon, you need to install the library in the virtual environment with the `py_install()` function. Finally, we use the created virtual environment with the `use_virtualenv()` function.



``` r
library(reticulate)

# create a python venv
virtualenv_create("scraping_telegram")
## virtualenv: scraping_telegram
```

``` r
# install the telethon library in the venv
py_install("telethon", envname = "scraping_telegram")
## Using virtual environment 'scraping_telegram' ...
```

``` r
# let reticulate use the venv
use_virtualenv("scraping_telegram", required = TRUE)
```


# Define the Python function

The Reticulate package allows you to execute Python chunks in R Markdown, which we will use to define a function that scrapes messages from a Telegram channel. This function takes a Telegram API ID and API hash ([see here](https://core.telegram.org/api/obtaining_api_id)), the channel name or url, the number of messages to scrape, and an offset date as arguments. This is just a simple example, and you can get much more data from the messages. In this case, the function returns a list with chat name, message ID, date, and message text. 



``` python
import telethon
import asyncio
from telethon import TelegramClient


# funtion to scrape messages
def scrape_messages(api_id, api_hash, channel, n, offset_date):
    
    # set up the Telegram client
    client = TelegramClient('session_name', api_id, api_hash)

    async def scrape(client, channel, n, offset_date):
      
        messages = []
        counter = 0
        
        async for message in client.iter_messages(channel, wait_time=3, offset_date = offset_date): 
            if not message.text:
                continue
              
            counter += 1
    
            raw_text = "" if message.raw_text is None else message.raw_text.replace("\'", "\"")
            
            messages.append({
            'chat_name': message.chat.username.lower(),
            'message_id': message.id,
            'date': str(message.date),
            'message_text': raw_text,
            })
            
            if counter >= n:
                break
              
        await asyncio.sleep(2) 
        return messages

    with client:
            messages_list = client.loop.run_until_complete(scrape(client, channel, n, offset_date))
    return messages_list
    
  
```

# Scrape messages

After defining the function, we can call it in R. Before that, we need to set the Telegram API ID and API hash. You can set them directly in the R script or use the `.Renviron` file to store them.




``` r
# set your credentials here or retrieve them from the .Renviron file
api_id <- 123456
api_hash <- "your_api_hash"
```

Now we can call the Python function directly with the `py$function()` syntax. For showcasing the scraping function, I will retrieve 5000 messages from the "CompactMagazin" channel, a right-wing conspiracy magazine in Germany. The default scraping order is from newest to oldest, and if you set an offset date, it will start scraping from that date down. At this point it can happen that the Telegram client asks you to authenticate yourself with phone number and 2-factor authentication code.




``` r
library(tidyverse)
# set the number of messages and the channel to be scraped 
n <- 5000
channel <- "CompactMagazin"

# with reticulate, you can call the python function directly
py_list <- py$scrape_messages(api_id,
                              api_hash,
                              channel,
                              n,
                              offset_date = as.Date("2022-05-28"))

```




The Python function returns a list, which we can convert to a data frame with the `bind_rows()` function from the `dplyr` package.

``` r
# convert the python list to a data frame
df <- dplyr::bind_rows(py_list)

glimpse(df)
```

```
## Rows: 5,000
## Columns: 4
## $ chat_name    <chr> "compactmagazin", "compactmagazin", "compactmagazin", "co…
## $ message_id   <int> 18382, 18381, 18380, 18379, 18378, 18377, 18376, 18375, 1…
## $ date         <chr> "2022-05-27 20:18:11+00:00", "2022-05-27 17:41:27+00:00",…
## $ message_text <chr> "💥UNZENSIERT💥Verbotenes Wissen über das Dritte Reich\n\…
```
So that's it. Scraping messages with Telethon via `reticulate` is quite easy. You can now use the scraped messages for further text analysis in R or network analysis.

# Simple text analysis
For Example, you can use the `quanteda` package to create a document-feature matrix and then use the great `stm` package for topic modeling. See [this](https://cran.r-project.org/web/packages/stm/vignettes/stmVignette.pdf) paper for more information on the stm package. 


``` r
library(stm)
library(quanteda)
```

Here is a simple example of how to create a structural topic model with the scraped messages. Before that, we need to create a pipeline to preprocess the text data and create a document-feature matrix. We only do minimal preprocessing here, but you can and should add more steps like stemming, lemmatization, or removing specific words. 
 


``` r
# create a document-feature matrix
dfm <- df %>% 
  mutate(date = as.Date(date),
         day = as.integer(date - min(date))) %>% # convert to days since start for use as covariate
  corpus(docid_field = "message_id",
         text_field = "message_text") %>% 
  tokens(
    remove_punct = TRUE,
    remove_symbols = TRUE,
    remove_numbers = TRUE,
    remove_url = TRUE,
    remove_separators = TRUE) %>%
  # remove stopwords
  tokens_remove(pattern = stopwords("de", source = "marimo")) %>%
  tokens_select(min_nchar = 3) %>%
  dfm(tolower = TRUE) 
```
We use the day of the message as a covariate to see how the topic prevalence change over time. In Topic Models like STM we need to specify the number of topics. There a several ways to determine the optimal number of topics, but for this example, we just use 10 topics, which is probably not the best choice. After creating the model, we can estimate the effect of the covariate day on the topic prevalence.


``` r
# convert dfm to stm format
out <- convert(dfm, to = 'stm')

# create a topic model
stm_model <- stm(K = 10,
                 prevalence = ~ day,
                 documents = out$documents,
                 vocab = out$vocab,
                 data = out$meta,
                 verbose = FALSE)

# estimate the effect of the covariate 
prep <- estimateEffect(1:10 ~ s(day), stm_model,
                            meta = out$meta)
```



Finally, our topic model is ready, and we can look into the results. As we have done minimal text pre-processing and have chosen an arbitrary number of topics, do not expect the topics to be very interpretable. For example, we can plot the topic prevalence over time of Topic 9, which seems to be about the "Russian Invasion of Ukraine".



``` r
plot(stm_model, type = "summary", xlim = c(0, 0.3))
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-12-1.png" width="672" />

For plotting the change of topic prevalence over time, we can use the `extract.estimateEffect()` function from the `tidystm` package. This function extracts the effects of the covariate from the model and returns a tidy data frame. We can then plot the topic prevalence over time with ggplot2.


``` r
library(tidystm)
# extract the effect of the covariate
effects_tidy_day <-  extract.estimateEffect(prep,
                                        covariate = "day", 
                                        model = stm_model,
                                        method = "continuous",
                                        labeltype = "prob")
```
The range of our 5000 Messages is from 2021-10-24 to 2022-05-27.

``` r
min(df$date)
```

```
## [1] "2021-10-24 06:51:41+00:00"
```

``` r
max(df$date)
```

```
## [1] "2022-05-27 20:18:11+00:00"
```
We create a sequence of days and join it with the effects_tidy_day data frame to plot the topic prevalence over time. We see that the topic prevalence of Topic 9 increases in the beginning of February 2022, a time when the first media reports about a possible Russian invasion of Ukraine appeared and peaked right after the invasion and stayed high until the end of the data collection period. This is not really surprising, but it shows how topic shifts can be modeled with STM. The most probable words of the topic are displayed in the plot caption.


``` r
# create the sequence
day_seq <- data.frame(date = seq(from = as.Date(min(df$date)),
                                 to = as.Date(max(df$date)), by = "day"),
                      day = min(effects_tidy_day$covariate.value):max(effects_tidy_day$covariate.value)) 

# join the effects dataframe
effects_tidy_day <- effects_tidy_day %>%
  mutate(day = as.integer(covariate.value)) %>%
  left_join(day_seq, by = "day")

# plot the topic prevalence over time
effects_tidy_day %>%
    filter(topic == 9) %>%
    ggplot(aes(x = date, y = estimate)) +
    geom_line() +
    geom_ribbon(aes(ymin = ci.lower, ymax = ci.upper), alpha = 0.2)  +
    theme_bw() + 
    scale_x_date(date_breaks = "month", date_labels = "%m/%y") +
    labs(x = NULL, y = 'Topic Proportion',
        title = paste("Topic 9" , "'Russian Invasion of Ukraine'"),
        caption = paste0("Words with highest probability:\n",
                         effects_tidy_day$label[effects_tidy_day$topic == 9]))
```

<img src="{{< blogdown/postref >}}index_files/figure-html/unnamed-chunk-15-1.png" width="672" />

I hope this post was helpful for you. If you have any questions, feel free to contact me. If there is something wrong with the code, please let me know. I am still learning Python and R, so I am happy about any feedback. 
