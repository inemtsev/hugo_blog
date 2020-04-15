---
title: "Chunked File Upload using Typescript, React, and Go"
date: 2020-04-15T19:30:11+07:00
draft: true
    priority: 0.5
---

### Summary - Why you may want to chunk files?
The biggest reason for me to upload files in chunks, is because I want to upload very large files; pictures, videos, whatever... This means, I want to know the status of the download as it progresses and if I cant finish the download now, I want to be able to pause, go to my favourite coffee shop and continue on there. 

<img src="https://aqzodowgen.cloudimg.io/bound/800x600/n/https://www.eventslooped.com/posts/img/chunked-file-upload-typescript-react-go/charisse-kenion-chunked-chocolate.jpg" alt="chunky chocolate, yum yum"/>
<a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px" href="https://unsplash.com/@charissek?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Charisse Kenion"><span style="display:inline-block;padding:2px 3px"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-2px;fill:white" viewBox="0 0 32 32"><title>unsplash-logo</title><path d="M10 9V0h12v9H10zm12 5h10v18H0V14h10v9h12v-9z"></path></svg></span><span style="display:inline-block;padding:2px 3px">Charisse Kenion</span></a>

To upload a file in chunks, we need to make our browser/client aware of the progress of the file upload. So, most of the logic for the chunking process will live on the client-side.
Let's start with a simple React component that currently has no logic but has a form that allows us to select multiple files. 

{{< gist inemtsev a45daa46fbdcdd6f80a65eed693a0689 "uploadMediaComponent.tsx" >}}

