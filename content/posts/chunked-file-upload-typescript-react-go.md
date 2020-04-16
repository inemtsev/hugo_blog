---
title: "Chunked File Upload using Typescript, React, and Go"
date: 2020-04-15T19:30:11+07:00
draft: true
    priority: 0.5
---

### Summary - Why you may want to chunk files?
The biggest reason for me to upload files in chunks, is because I want to upload very large files; pictures, videos, whatever... This means, I want to know the status of the download as it progresses and if I cant finish the download now, I want to be able to pause, go to my favourite coffee shop and continue on there.

### Let's build a simple app using no additional javascript libraries! 

<img src="https://aqzodowgen.cloudimg.io/bound/800x600/n/https://www.eventslooped.com/posts/img/chunked-file-upload-typescript-react-go/charisse-kenion-chunked-chocolate.jpg" alt="chunky chocolate, yum yum"/>
<a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px" href="https://unsplash.com/@charissek?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Charisse Kenion"><span style="display:inline-block;padding:2px 3px"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-2px;fill:white" viewBox="0 0 32 32"><title>unsplash-logo</title><path d="M10 9V0h12v9H10zm12 5h10v18H0V14h10v9h12v-9z"></path></svg></span><span style="display:inline-block;padding:2px 3px">Charisse Kenion</span></a>

To upload a file in chunks, we need to make our browser/client aware of the progress of the file upload. So, most of the logic for the chunking process will live on the client-side.
Let's start with a simple React component that currently has no logic but has a form that allows us to select multiple files. 

{{< gist inemtsev a45daa46fbdcdd6f80a65eed693a0689 "uploadMediaComponent0.tsx" >}}

Let's create a class that will keep track of all the details about the file and its progress; we will also put all the methods we need inside. 

{{< gist inemtsev  a45daa46fbdcdd6f80a65eed693a0689 "FileToUpload0.ts" >}}
What's inside? **chunkSize** is the size of each chunk you wish to send to the server. **uploadUrl** is the path to your server-side endpoint (more on that later). **file** simply refers to the file you have selected to upload. Getting to where all the magic happens, **currentChunkStartByte** is where we keep track of where the starting point of the current chunk is located. **currentChunkFinalByte** is where we keep track of the location of the final byte of the current chunk. 

Inside the constructor we instantiate the browser built-in **XmlHttpRequest** class and set the Mime Type to **application/octet-stream**, which simply refers to generic binary data. **currentChunkFinalByte** is set to either the chunkSize or the file size, whichever is smaller. This is to avoid chunking of small files. 

Let's add the method that will do all the uploading. 

{{< highlight ts >}}
uploadFile() {
        this.request.open('POST', FileToUpload.uploadUrl, true);

        let chunk: Blob = this.file.slice(this.currentChunkStartByte, this.currentChunkFinalByte);  // split the file according to the boundaries
        this.request.setRequestHeader('Content-Range', `bytes ${this.currentChunkStartByte}-${this.currentChunkFinalByte}/${this.file.size}`);
        
        this.request.onload = () => {
            const remainingBytes = this.file.size - this.currentChunkFinalByte;
            
            if(this.currentChunkFinalByte === this.file.size) {
                alert('Yay, download completed! Chao!');
                return;
            } else if (remainingBytes < FileToUpload.chunkSize) {
                // if the remaining chunk is smaller than the chunk size we defined
                this.currentChunkStartByte = this.currentChunkFinalByte;
                this.currentChunkFinalByte = this.currentChunkStartByte + remainingBytes;
            }
            else {
                // keep chunking
                this.currentChunkStartByte = this.currentChunkFinalByte;
                this.currentChunkFinalByte = this.currentChunkStartByte + FileToUpload.chunkSize;
            }

            this.uploadFile();
        }

        const formData = new FormData();
        formData.append('file', chunk, this.file.name); 
        this.request.send(formData);    // send it now!
    }
{{< / highlight >}}
A few things to highlight here. **Content-Range** needs to be set, we will use this on the server to determine when to stop adding to the file. **this.request.onload** is a callback which is called whenever a transaction by XmlHttpRequest is completed. [More on XmlHttpRequest.onload here](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequestEventTarget/onload).

Putting these together, your **FileToUpload.ts** will look like this.
{{< gist inemtsev  a45daa46fbdcdd6f80a65eed693a0689 "FileToUpload.ts" >}}

### Let's use what we created inside our component

Now, that we have a class to represent the file to be uploaded. We can instantiate this class as many times as needed inside the component. The resulting component looks like this.

{{< gist inemtsev a45daa46fbdcdd6f80a65eed693a0689 "uploadMediaComponent.tsx" >}}

### Our client-side code is ready to go, let's prepare the backeng using Go and Gin

