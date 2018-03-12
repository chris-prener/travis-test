# travis-test
[![Travis-CI Build Status](https://travis-ci.org/chris-prener/travis-test.svg?branch=master)](https://travis-ci.org/chris-prener/travis-test)
[![AppVeyor Build Status](https://ci.appveyor.com/api/projects/status/github/chris-prener/travis-test?branch=master&svg=true)](https://ci.appveyor.com/project/chris-prener/travis-test)

This is a proof of concept for testing `R` scripts on continuous integration services.

## Motivation
As I noted on a post in the [RStudio Community boards](https://community.rstudio.com/t/package-surprises/5000), I had a few speed-bumps earlier in my Spring semester while teaching [Introduction to GIS](https://slu-soc5650.github.io). My laptop and desktop, both Apple devices are my primary "testing" machines for `R` packages and processes that I teach with. However, I teach in a Windows lab and the majority of my students use Windows on their personal computers. This occasionally leads to issues where the Windows process for installing or using an `R` package differs in some slight way. 

I try and test code on Windows computers when I can (I have a virtual Windows partition on my iMac, but this is only in my office so I can't test code on Windows if I am not on campus), but my teaching days are hectic and I don't always have the free time I wish I had. Testing on Windows when I do have time has also not been helpful in situations where the software that my students are installing has changed in some way from the installed versions that I already have. This happened this semester with `janitor`, where the package was updated between my own testing and when my students installed it.  (No shade at `janitor` - I love the updates - it just caught me by surprise!)

The issue here, for me, is not that packages are updated and sometimes unstable. Learning to manage that is part of learning to use `R`. I teach graduate students and (typically) advanced undergrads, and while there is a lot of [command-line bullshittery](http://pgbovine.net/command-line-bullshittery.htm) to navigate, I think that it is important to give students the capability to get `R` up and running locally. If I taught first year undergrads, I might feel differently. 

Rather, the issue for me is being prepared for how that installation process is going to go on various machines and making sure that updates to packages do not break the examples that I am teaching with. I teach `R` each semester, but in different settings - statistics in the fall and Intro to GIS in the spring. That means that my examples sit for nearly a year between courses. This feels like a near eternity in the `R` community, especially when I am teaching with packages undergoing active development like `sf`, `naniar`, and `janitor`. 

## A Solution?
RStudio community user [Peter Gensler](https://community.rstudio.com/t/package-surprises/5000/2?u=chris.prener) suggested thinking of teaching code as software, and using a continuous integration service. Awesome idea! What I like about this is that I can get fresh installs of `R` and the packages that I use, and test code at will rather than having to wait until I have access to my office computer's virtual Windows machine. This also allows me to automatically check the code that I am running to make sure it has not been broken by package updates. This approach also allows me to check my code on various versions of `R`, which is important given that my students are not always using current software (despite my best efforts!). Finally, I like this approach because it also checks code to make sure that I haven't broken it myself if I update one part of a script but not another.

## Needs
In short, my needs with using continuous integration are:

* To be able to run fresh installations of packages in a controlled way that allows me to catch issues
* To be able to test code used in a previous iteration of the course to make sure that package updates do not break any examples that I use in class

In addition, I had a couple of other needs that I wanted to be able to speak to with the testing process.

### Focus on Notebooks
For both my courses and other seminars I give on campus, I use `R` notebooks to introduce students to literate programming, Markdown, and `knitr` in addition to `R` itself. I provide notebooks as examples, and students submit the `R` portions of assignments as notebooks as well. Since I am already "shipping" my lecture repositories with notebooks, knitting these notebooks seemed to be an easy entry point into testing code for my courses.

### Focus on Windows and macOS
I wanted to focus my testing on Windows and macOS for two reasons: (1) none of my students use Linux and (2) I teach with `sf`, especially in the Fall, and getting `sf` to install on Linux is... a pain. This focus means getting the process up and running both on Travis-CI and Appveyor. 

### Installing from GitHub
I ship data in packages that are not available on CRAN (and in some cases may never be). Setting-up the testing meant that I needed to be able to make this happen remotely as well.

## Implementation
After a ton of trial and error (including way too much of my Sunday evening devoted to getting `sf` to install on Linux, which was ultimately unsuccessful), I have a working implementation that hits all of the high points that I want to include in continuous integration for teaching code.

### Using a `DESCRIPTION` File
I added a `DESCRIPTION` file to my test repository following [Hadley Wickham's advice](https://github.com/travis-ci/travis-ci/issues/5913). I took out a number of the pieces that were specific to `R` packages. The key part here is to specify the packages that I will be teaching with for a given lecture. This serves as a way to instruct both Travis and Appveyor to install the needed packages. My `DESCRIPTION` file ensures that `knitr`, `rmarkdown`, and `sf` are all dependencies for this test lecture.

### Travis-CI
For Travis, I disabled caching to force my builds to re-download software each time. This slows down the process but ensures that I am able to catch any installation errors each and every time I build a lesson's code. I also specified that builds only run on macOS, and that they run on both the old release and current release of `R`. The top of my `.travis.yml` file therefore looks like this:

```yml
language: r
sudo: false
cache: false

r:
  - oldrel
  - release

os:
  - osx
```

To install packages from GitHub, I included the following in my `.travis.yml`:

```yml
r_github_packages:
  - chris-prener/stlData
  - tidyverse/ggplot2
```

I am installing the dev version of `ggplot2` since my Intro to GIS course uses it for `geom_sf()`, and I am installing one of my own data packages that I use for examples in class.

Finally, to test my scripts, I include the following in my `.travis.yml`:

```yml
script:
  - R -e "rmarkdown::render('tests/test.Rmd')"
  - R -e "rmarkdown::render('tests/testMap.Rmd')"
```

### Appveyor
For Appveyor, I removed the caching altogether from the `appveyor.yml` file for the same reason that I disabled it on Travis - I want to install everything fresh each time even if my builds will take longer. I ran into some early issues with Pandoc not being installed at a version that `knitr` needed, but luckily I was able to find [this issue](https://github.com/krlmlr/r-appveyor/issues/82) where Jenny Bryan offered an installation solution. Her code is included in my `appveyor.yml` file:

```yml
before_test:
  - ps: >-
      if (-Not (Test-Path "C:\Program Files (x86)\Pandoc\")) {
        cinst pandoc
      }
  - ps: $env:Path += ";C:\Program Files (x86)\Pandoc\"
  - pandoc -v
```

I also specify a build matrix that includes a few different variations of Windows and `R`. As with Travis, I forgo testing on the development version of `R`:

```yml
environment:
  global:
    WARNINGS_ARE_ERRORS: 1

  matrix:
    - R_VERSION: patched
      R_ARCH: x64

    - R_VERSION: patched

    - R_VERSION: release
      R_ARCH: x64

    - R_VERSION: release

    - R_VERSION: oldrel
      R_ARCH: x64

    - R_VERSION: oldrel
```

To install both the dependencies and the two packages from GitHub, I include the following in `appveyor.yml`:

```yml
build_script:
  - travis-tool.sh install_deps
  - travis-tool.sh install_github chris-prener/stlData
  - travis-tool.sh install_github tidyverse/ggplot2
```

And finally, to test my scripts, I replace the default package testing with the following:

```yml
test_script:
  - Rscript -e "rmarkdown::render('tests/test.Rmd')"
  - Rscript -e "rmarkdown::render('tests/testMap.Rmd')"
```

## Final Thoughts
In testing this system out on this repository, I've both increased my own knowledge of how Travis and Appveyor work (which I am super stoked about!) and come up with a sustainable way to test my code that works for how I teach. If you don't teach with notebooks, this process will take some alteration. I'll be rolling this out in my courses next academic year as well as in the [SLU Data Science Seminars](https://slu-dss.github.io) that I help organize, and will put together a more formal blog post then!