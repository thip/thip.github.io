---
layout: post
title:  "Building a Vegetable Encyclopedia Using GPT-4 and DALL-E"
image: /assets/robot_with_vegetables.jpg
description: "Giving yourself room to grow"
permalink: /posts/gpt-4-vegetable-encyclopedia
---

![Robot holding vegetables](/assets/robot_with_vegetables.jpg)
*Photo by Dall-E 2*

## Introduction

I recently embarked on a fascinating journey to build a website that serves as an A-Z of vegetables. This website includes descriptions, facts, history, and month-by-month growing advice for each vegetable. I wanted to see how far I could push the boundaries of the AI world and decided to use GPT-4 and DALL-E from OpenAI to power this project. In this blog, I'll walk you through my experiences and the steps I took in using these cutting-edge AI technologies to create a static website generator which can be found at [https://vegetables.hortus.dev](https://vegetables.hortus.dev).

## Step 1: Getting the Data

I started by using the chat interface on [https://platform.openai.com/playground](https://platform.openai.com/playground) and the official API to interact with GPT-4. I asked it to act as a JSON API and gave it a schema to fill out for various vegetables. This provided me with a structured JSON dataset to work with.

## Step 2: Building the HTML Page

Next, I fed GPT-4 one of the JSON responses and asked it to create an HTML page that would display the information in a user-friendly format. After some back-and-forth discussions and edits with GPT-4, I had a page that I was satisfied with.

## Step 3: Creating a Golang Template

To make the website dynamic, I asked GPT-4 to convert the HTML page into a template for the Golang programming language. This allowed me to effortlessly create new pages with different content using the same layout.

## Step 4: Generating a Web Application

I then instructed GPT-4 to write a program that would accept a directory of JSON files and populate the pages with the template. This made it simple to generate various pages for each vegetable in the list.

## Step 5: Building the Directory Page

To allow users to navigate the website easily, I asked GPT-4 to create a directory page which would serve as an index for all the vegetable pages.

## Step 6: Adding DALL-E-generated Images

To enhance the visual appeal of the website, I wrote a script to prompt DALL-E for images of each vegetable. Some of these images were a bit peculiar, and so I had to regenerate them until I got a set I was pleased with.

## Step 7: Updating the Website with Images

Finally, I asked GPT-4 to add the images to the index page as cards and display each entry with its corresponding image. I also updated the individual vegetable pages with the new images using the Golang program.

## Deployment and Hosting

As the website generator now creates a static website, I used GitLab Actions to automate the process of building the HTML files and copying them into a directory. I then served the website on GitLab Pages. This deployment pipeline can also be easily adapted for use with AWS and Amazon S3.

## Conclusion

Using GPT-4 and DALL-E to build this vegetable encyclopedia was an incredibly satisfying experience. Working with GPT-4 felt like collaborating with an intelligent junior engineer. While it sometimes encountered roadblocks or made unexpected decisions, it was easy to guide and correct through a conversational approach.

In the end, the project cost me about $20 for the API calls, mostly from the interactive discussions with GPT-4. However, the results were worth every penny, and I'm delighted with the final product. This experiment proved that AI-driven tools like GPT-4 and DALL-E hold immense potential in making web development more accessible, efficient, and creative than ever before.

(In case you didn't realise, this entire post was written by GPT-4 too!)