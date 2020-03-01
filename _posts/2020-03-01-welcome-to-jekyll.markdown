---
layout: post
title:  "How I run R code"
date:   2020-03-01 19:50:32 +0100
categories: R
---
### R on the server

Most of the biological datasets(beyond the toy examples) require considerable computational resources.
That's why we need to run our code on powerful servers. But if you have only worked in
R Studio before, you'll likely wonder: "How do I run my R code on the server?". The first instinct might be to `ssh`
to the server, fire up `R` console and start copy-pasting the code line-by-line from your local computer. This might work
well if you're experimenting with your dataset. But what if you need to repeatedly re-run the same code or the computations take hours? Moreover, such naïve approach jeopardizes reproducibility: you might forget running one line of the code and then you never know how exactly have you obtained your results.

Whenever you tweak some parameter in your pipeline, you need to be able to re-run everything with the minimum effort.

### Good practices

I have come up with very simple patterns I apply to all my scripts. Let's dive in.

#### Running R script from the command line

The first one is quite easy. Suppose you have developed a script called `exportData.R`. On Linux or Mac you'd simply run:

{% highlight bash %}
Rscript exportData.R
{% endhighlight %}
This will execute the script and print the output in the terminal.

#### External configuration

One of the most important principles of software engineering is separating your code from the configuration. What do I mean by that? Suppose you have the following code in the `exportData.R`:

{% highlight R %}
library(tidyverse)

data <- read_csv("/home/f6v/rna_seq_counts.csv")
{% endhighlight %}

Your script is loading `rna_seq_counts.csv` file from `/home/f6v/` directory on the server. What happens when the file name changes? You'll need to edit your code, which can become tedious and error-prone in practice. The solution is to provide *external* configuration to your script. When running your script, your can also provide input arguments.

{% highlight bash %}
Rscript exportData.R export_data_config.yaml
{% endhighlight %}

`YAML` files are a good choice for storing configuration, since they are designed to be human-readable. The `export_data_config.yaml` should then be created on the server and might look like this:

{% highlight yaml %}
data_path: "/home/f6v/rna_seq_counts.csv"
{% endhighlight %}

And in your script:

{% highlight R %}
library(tidyverse)
library(yaml)

config_file_name <- commandArgs(trailingOnly = TRUE)[1]
config <- yaml.load_file(config_file_name)
data <- read_csv(config$data_path)

{% endhighlight %}

The code above takes the first argument which you provided when running the script, loads yaml file and supplies `config$data_path` as an argument to `read_csv`.

This might seem like a silly example, but often you'd have many parameters when processing your data. For example: number of CPU cores(which might change based on current server load), which subset of data to use(i.e. only one of the chromosomes if you're working with sequencing data), etc. Moreover, you can test your script on the subset fo data locally first, and then push it to server, all without changing the code itself, but rather having different configuration files locally and on the server.

#### Pushing the code to the server

How do you actually get your code to the server? This is quite simple to do with `scp` utility. You can run it from your local machine from the folder which contains the code(if you're on Linux or Mac):

{% highlight bash %}
scp -P 2222 -r . f6v@server.address.com:/home/f6v/code
{% endhighlight %}

This will push the code from my local machine to the `/home/f6v/code` on the server. If you're on Windows, you can use MobaXterm.


#### Long-running scripts

If you work with big dataset, it might take your script hours to process the data. And the thing with running script from Unix terminal is that the code will run only as long as you're connected to the server. If you decide to close your laptop, or simply loose Wi-Fi signal, you'll have to start all over again. There're several solutions to this problems. I prefer using `screen` utility which is likely already installed on your server. I won't go into too much details(as there're many tutorials already), but here's the gist. You log in to the server, and run:

{% highlight bash %}
screen -S my_data_processing
{% endhighlight %}

This creates a `screen` which will be alive even if you disconnect from the server. You can run your script as usual in the screen, and then press `Ctrl-A` followed by `Ctrl-D`. After that you can safely disconnect, and your script will keep on running. Next time you connect to the server, simply run:

{% highlight bash %}
screen -r my_data_processing
{% endhighlight %}

Which will bring you back to the screen named `my_data_processing`. I usually leave my data pipelines running overnight and check on them in the morning, and I don't have to keep my laptop running.


#### Logging

To follow up on the last paragraph, I often want to see how long certain script took to run if I left it overnight. Also if the script processes large amounts of data, I want to see the progress. I think it's terrible if you're running script for 8 hours and have no idea what's happening. Is it processing data, or did it just hang?

Those are good examples of *logging*. It's a good practice to save diagnostic information to the log file, which you can examine afterwards. For example, if I process sequencing data I save start and end time for each chromosome. If my script crashes, I can see on which chromosome it crashed, which helps with debugging. I can also see how long did it exactly take for my script to run and decide if I need to optimise the code.

In R you can easily log with the `futile.logger` package. The following code logs the message to the `data_export.log` file.

{% highlight R %}
library(futile.logger)

flog.logger("data_export", INFO, appender = appender.file('data_export.log'))
# Some other code
flog.info('Started processing for chromosome %s', current_chromosome, name="data_export")
{% endhighlight %}

#### Wrap up

The practices I've described save me a lot of time and help troubleshoot the issues quicker. Next time I'll describe how you can easily parallelise your code.
