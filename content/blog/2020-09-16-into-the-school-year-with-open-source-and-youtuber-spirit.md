---
title: Into the new School Year with Open Source and YouTuber Spirit 
slug: into-the-school-year-with-open source-and-youtuber-spirit
summary: "Instead of doing the same old bad slides combined with bad audio on Zoom, let's learn from YouTubers and dig into free and open source tools to change the way we teach in the face of the pandemic."
authors:
- janos
categories: blog
images:
- /posts/into-the-school-year-with-open-source-and-youtuber-spirit/social.png
tags:
 - education
date: "2020-09-16T00:00:00Z"
publishDate: "2020-09-16T00:00:00Z"
---

Late last year a colleague of mine offered me his lecturer position at [a local university of applied sciences](https://www.fh-campuswien.ac.at/en/index.html). Little did I know, my previous experience in teaching online would come in *very* handy as we were heading into a whole world suddenly switching to remote education.

The initial idea was only to bring a more practical approach to the lectures. However, by the time we went into the first round of discussions with the university leadership we were already in lock-down mode, and the idea of replicating remote education from the past became very appealing.

In this post I will showcase the tools used to create [this cloud computing lecture](https://fh-cloud-computing.github.io/) and explain how you can use them to create a similar experience for your students. 

## The concept

With the lock-down everything changed. Students could no longer go into class rooms and pretend to listen to lecturers. Making a list of who's present no longer made sense. Splitting a class room into groups for a project was no longer possible. Sure, Zoom added breakout rooms, but the [universally bad audio quality](https://www.youtube.com/watch?v=JMOOG7rWTPg) made these either outright impossible, or at least very painful to listen to.

Let me demonstrate what I mean. Please listen to the following audio sample (or [download it](/posts/into-the-school-year-with-open-source-and-youtuber-spirit/reverb-demonstration.mp3)):

<audio preload="none" src="/posts/into-the-school-year-with-open-source-and-youtuber-spirit/reverb-demonstration.mp3" controls></audio>

Do you see what I mean? In a class room your brain naturally deals with the echo as you are in the same setting. When listening to someone on your computer you are not. This makes it a lot harder to understand what's being said, and you are using your brain power to understand the words rather than comprehend the meaning.

The way we worked changed, too. Suddenly, nobody cared if you were cooking lunch, driving in a car, or taking care of your kid while being in a meeting. It became socially acceptable to be more efficient. It was suddenly OK to participate in a lecture while driving somewhere in a car. 

Does it really make sense to *force* people to be there at the same time and look at the screen, even when nothing is happening? Does it really make sense to have a classic lecture where there's a *lot* of downtime?

Let's play a thought experiment. Or rather, let's replicate an experiment I was confronted with back in 2018: you have the task to educate a large number of people who are distributed all over the continent. Furthermore, this group of people is constantly changing since people are hired, or leave the company from time to time. How would you do it?

If you think about it, this is exactly what online learning platforms are doing. Most platforms focus on video only. All the corporate platforms I have seen usually include material that nobody subjects themselves to because it's just so fundamentally boring. The lecture is recorded in the same way a university lecture would: lots of downtime, bad audio quality.

This is just the *millenial impatience* at work, at the computer we are lacking a lot of non-verbal queues. It is a lot harder to listen to a disembodied, echo-y voice of a teacher than it is to sit in a class room. Also, the teacher can't read our body language and stop if we don't understand something.

In contrast, there are millions of YouTubers, Udemy, or Linkedin educators out there, who want to do exactly that. Their videos are tightly edited together to provide a great learning experience. A great learning experience *if you want a video*. So, that's part of the solution.

However, not all of us want a video. When learning for a test we *skim* material very quickly. Or, we print it out and use a marker to highlight the important bits. Also, some people are just simply better at ingesting information by reading than they are by listening to someone. We also want to support the scenario when someone wants to consume information *on the go*.

We have two goals here: provide content for different *purposes*, and helping people of [the 4 learning styles](https://www.rasmussen.edu/degrees/education/blog/types-of-learning-styles/).

We have all the tools in our hand: we are already making slides, a lot of university teachers are already writing notes, and we use our vocal chords to produce sound. We already have everything needed for **multi-format educational material**.

This multi-format educational material would encompass a *text* version of the material to be learnt. It would also contain illustrative (not bulled point) slides to help the visual types. It could then also have an *audiobook* for auditory learning, and finally combine the slides, and the audiobook to create a video.
Furthermore, we can also add our previous bullet point list to the slide speaker notes, and to the text version as well. If we really want to go out of our way we can add a quiz for the tactile learners. 

## The text version

Let's start with the text version. One of the open source tools I really like is [mkdocs](https://www.mkdocs.org/), a popular tool for software documentation. Together with the [material theme](https://squidfunk.github.io/mkdocs-material/) it is capable of accommodating larger amounts of text. It also has built-in facilities for creating highlighted boxes for tips, tricks, and warnings called [admonitions](https://squidfunk.github.io/mkdocs-material/reference/admonitions/).

mkdocs allows us to create documentation in an easy to edit format called [markdown](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet). Unlike Microsoft Word, Markdown is a purely text format and is easy to handle and transform when needed. Most importantly, mkdocs can make a great looking website out of it.

Once it is set up less technical people can be taught to edit content in it too. [GitHub](https://github.com/) offers everything needed to edit, and to publish the website. (Yes, they provide free hosting.)

The initial setup is not *too* difficult, but you will need a bit of technical finesse. If you haven't already, head on over to [GitHub](https://github.com/) and register a free account. Once you're done, go to [my template repository](https://github.com/janoszen/mkdocs-blueprint) and click the green &ldquo;Use this template&rdquo; button. This will give you an option to make a copy of my prepared mkdocs setup into your own account. **Make sure to check the &ldquo;Include all branches&rdquo; option.**

Once the process is complete, move to the &ldquo;Actions&rdquo; tab and wait for the deployment to complete. Once that's done, go to &ldquo;Settings&rdquo; and scroll down to the &ldquo;GitHub Pages&rdquo; section. Please select `gh-pages` as the source and click &ldquo;Save&rdquo;.

![](/posts/into-the-school-year-with-open-source-and-youtuber-spirit/github-pages.png)

Congratulations! **Wait a few minutes**, and you will be the owner of a brand new e-learning website located at `https://YOURUSER.github.io/YOURREPONAME/`! If you wish, GitHub will give you the option to set up a custom domain name for your website too. You can read more about that [in the official documentation](https://docs.github.com/en/github/working-with-github-pages/configuring-a-custom-domain-for-your-github-pages-site).

Now, on to customizations. Please look at your GitHub repository and open the `mkdocs.yaml` file. This is your main configuration file in the popular [YAML format](https://rollout.io/blog/yaml-tutorial-everything-you-need-get-started/). You can click the little pen icon to edit it. Don't worry, GitHub keeps your old versions in case you mess up.

Once you are done customizing everything there, head on to the `docs` directory. This is where your content lives. Primarily, in the files titled `index.md`. The examples provided should show you the way. You can now add content to your heart's content (pun totally intended).

You can also add admonitions like this:

```
!!! note
    This is a note

!!! tip
    This is a tip

!!! warning
    This is a warning
```

Or you can add a quiz:

```
{{ quiz("What components are redundant in a server?", [
    answer("Power supply"),
    answer("CPU"),
    answer("RAM"),
    answer("Fan"),
    answer("OOBM"),
]) }}
```

The quizzes don't give the students feedback because they are designed to invoke thinking rather than giving them an easy to memorize answer.

## Adding audio

So far you only have a text version of your course. Let's add some audio. You can record audio with a great many tools, one of the most popular open source ones is [Audacity](https://www.audacityteam.org/). I personally use [Adobe's Audition](https://www.adobe.com/products/audition.html) for this purpose.
 
 Besides the audio software, you will also need a microphone. Nothing fancy, even a basic headset will do as long as the microphone is close to your mouth. **Do not use in-ear headphones (e.g. Apple AirPods) or your built-in laptop microphone.**

You will need to find a quiet room. The room should also not be empty as empty walls reflect sound and cause reverb in the audio. Now comes the hardest part: you will need to read the text and listen to your own voice. *A lot.* The hard part here is not to fall into a monotone reading voice as most of us tend to do when we are sitting alone. If you are having a hard time doing this get someone else you can read to.

Also, make sure you experiment with your setup before you do the whole read. Move your microphone position to see what works best and how you can get the best result. When you are done with reading check your audio program if you can auto-remove long pauses. If you want to experiment with noise removal make sure you don't over do it as it results in very weird and hard to listen-to artifacts.

Once you have your audio recorded, each segment in its own file, you can upload them to GitHub. Then you can add the audio to your `index.md` file by inserting the following HTML code below your headlines:

```html
<audio preload="none" controls src="your-audio-segment.mp3"></audio>
```

If you add a separate file after each headline, the blueprint will take them and chain them together for listening. If you want to add a downloadable MP3, too, you can merge the files together in your audio editing program and add them to the top of the page.

## Creating a slide deck

The next step is to create the slide deck. Traditionally, slide decks contain lots of bullet points. That serves a purpose as the slide deck is what students learn from. In our case, however, the slide deck is intended to invoke memorization. Memorization is helped by *unique visuals*. In my experience, most slides should contain a graphic relating to the content that's being talked about.

![A picture showing a slide with colorful graphics.](/posts/into-the-school-year-with-open-source-and-youtuber-spirit/visual-slides.png)

Ideally, you'll have one slide for every paragraph of text. However, that is a crazy amount of work, so I stuck with creating somewhere between 1 and 5 slides per section. These slides will later serve the purpose of video as well.

But, before we start creating slides like crazy let's make sure the slides can be exported in the correct resolution for video. If you are using Microsoft Office head on to Design &rarr; Slide Size and select &ldquo;Custom Size&rdquo;. Change the slide size to 20 inches by 11.25 inches. This will set your output size to 1920x1080. If you don't do this the slides will be saved as  960x540 with no easy way to change it.

If you are on the Libre(Office) side of things you are in luck: it will ask you for the intended target resolution of your files. The only thing you have to make sure is that your slides are in <span role="text" aria-label="16 by 9">16:9</span> format and not <span role="text" aria-label="4 by 3">4:3</span>. (Seriously, Microsoft, can we have a resolution option please?)

Once you are done with your slides, export them to individual PNG files. This will be used when creating the video.

If you want to provide your students with the slides make sure to add the text to the speaker notes of each slide. Students can use the notes view to create printouts with visual aids to help them learn. You can save and upload your PPTX to GitHub as well and link them at the top of the page. Just make sure to select the &ldquo;compress images&rdquo; option and set the resolution to 300 DPI as GitHub allows a maximum file size of 100 MB for a single file.

## Creating the video

A final piece to the puzzle, and your career as a potential YouTube educator is creating the video. First, collect your slides in PNG format and your audio snippets into a folder. This will be the basis for creating the video. Next, you will have to pick a video editing software. I'm using [Adobe Premiere Pro](https://www.adobe.com/products/premiere.html) as I have the Adobe suite, but if you want to stay open source you can try [OpenShot](https://www.openshot.org/). I have used it to great success in the past. The steps will be similar in any video editor.

First, pull your audio files into the audio track in order. Then grab your PNG files and drag them into the video track. Finally, synchronize the two. I found this easiest to do by adding one extra slide per headline using the wonderful pictures from [Unsplash](https://unsplash.com/). This will make synchronizing a set of slides with an audio segment easy, and also give you cue points when you upload the video to YouTube.

![A demonstration of the PNG files correctly aligned with the audio segments in Premiere.](/posts/into-the-school-year-with-open-source-and-youtuber-spirit/video-editing.png)

Once you are done export the video in h264/MP4 format with the resolution of 1920x1080, and a frame rate of 29.97 frames per second. Make sure you use sqare pixels rather than anamorphic ones if that is an option.

Finally, head on over to [YouTube](https://www.youtube.com/) and upload your video. While your video is uploading, copy the raw text into a plain text file (e.g. by using Notepad). Once the video has finished uploading you can upload this text file for subtitles and YouTube will automatically synchronize it to your spoken text. This is a nice way to automatically generating SRT files for subtitles even if you don't use YouTube as your final platform.

If you are using YouTube, you have one more option. Edit the description of your video and add the timestamp for each segment in the following format:

```
0:00 Introduction
1:52 Second chapter
``` 

This will add chapter navigation to the video so your viewers have an easier time finding individual pieces of content.

![](/posts/into-the-school-year-with-open-source-and-youtuber-spirit/youtube-video-chapters.png)

## Accessibility

Having closed captions already hints in this direction: accessibility is important. It's not just people with visual disabilities you should think of, but people with motor control impairments, or simply *people on their mobile phone*.

Yes, it's surprising. Many of the aspects that make a website accessible also happen to make them more mobile friendly. If you have poor text-to-background contrast your website will not be readable on a mobile phone in bright sunlight, for example.

Here's a few things to watch out for:

- Is your text readable if you are on a mobile device in bright sunlight? If not, add more contrast to the text.
- Do your images convey additional context? If yes, do you have an [alternative text](https://en.wikipedia.org/wiki/Alt_attribute) that explains the image to those using a screen reader?
- Do your images have a reasonable size or did you put in gigantic images? Large images may pose problems for students with data caps or slow internet.
- Is your text readable with screen readers? Try it out by enabling screen reading on your mobile device!

## Project work

The parts above cover the theory section of our material. Our students also have to hand in a project work over the course of the semester.

We structured the project work into 3 segments, each lasting one month. We did this to ensure that the students don't leave handing in their work to the last second. We have also decided to go for individual work instead of group submissions as coordinating a group effort remotely is more complicated for students. Remember, even though our course is a master's course in IT, the students are still not used to working together remotely in an efficient manner.

In order to help the students with the project we also [provided exercises](https://fh-cloud-computing.github.io/exercises/).

We have considered providing a video version next to the text version, but dismissed this idea. The reason we dismissed the video idea is for educational reasons. Our students will have to read and execute instructions accurately, and use them as building blocks for larger architectures later in their professional life.

The exercises provide them with the bricks for the project work they need to hand in. They will need to put the building blocks together themselves and do additional research to fill in the gaps. 

## Helping students

Creating course material is awesome, but not enough. In a class room setting you can read your students' body language and see if they understand your lecture immediately. You can give additional context if needed, or skip over trivial parts if everyone looks bored. However, in an online setting, even in Zoom, you don't have these nonverbal cues.

In our case, giving students modern and efficient channels to interact with us and fellow students was critical. We decided to replace the in-person interaction with regular *consulting* calls where students could ask questions closely related to the course material, or about the wider industry in general. Additionally, we have created a [Slack channel](http://slack.com/) so students can exchange information and ask questions interactively.

## Final words

As you might imagine, creating real online learning material is a magnitude more work than slapping some bullet points and a few images copied from the Internet on a slide deck and hopping on a Zoom call. It is more akin to writing a book in volume. I spent well over a 100 hours writing our course material.

However, like books, this material has a certain longevity. Students can consume the content in their own pace, on their own time. You will not have to be there and hold the same lecture over, and over, and over again. What you will have to do is help your students fill in the gaps and get over the hurdles. In other words, your time will also be put to better use this way.