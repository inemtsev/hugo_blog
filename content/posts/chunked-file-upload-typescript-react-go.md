---
title: "Chunked File Upload using Typescript, React, and Go"
date: 2020-04-15T19:30:11+07:00
draft: false
sitemap:
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

### Our client-side code is now sending chunks, let's prepare the backend using Go and Gin

<img src="https://aqzodowgen.cloudimg.io/bound/800x600/n/https://www.eventslooped.com/posts/img/chunked-file-upload-typescript-react-go/sven-brandsma-chainsaw.jpg" alt="vroom vroom"/>
<a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px" href="https://unsplash.com/@seffen99?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Sven Brandsma"><span style="display:inline-block;padding:2px 3px"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-2px;fill:white" viewBox="0 0 32 32"><title>unsplash-logo</title><path d="M10 9V0h12v9H10zm12 5h10v18H0V14h10v9h12v-9z"></path></svg></span><span style="display:inline-block;padding:2px 3px">Sven Brandsma</span></a>

Starting with the typical Gin setup, we add one endpount **/photo**

{{< gist inemtsev a45daa46fbdcdd6f80a65eed693a0689 "main0.go" >}}

Then we handle the request:

{{< highlight go >}}
func uploadFile(c *gin.Context) {
	var f *os.File
	file, header, e := c.Request.FormFile("file")

	if f == nil {
		f, e = os.OpenFile(header.Filename, os.O_APPEND|os.O_CREATE|os.O_RDWR, os.ModeAppend)
		if e != nil {
			panic("Error creating file on the filesystem: " + e.Error())
		}
	}

	if _, e := io.Copy(f, file); e != nil {
		panic("Error during chunk write:" + e.Error())
		f.Close()
	}

	if isFileUploadCompleted(c) {
		uploadToGoogle(c, f)
		if e = f.Close(); e != nil {
			panic("Error closing the file, I/O problem?")
		}
		if e = os.Remove(header.Filename); e != nil {
			panic(fmt.Sprintf("Could not delete file %v", header.Filename))
		}
	}
}
{{< / highlight >}}

The logic here is fairly straightforward:

1. Open the file or create it if not there. **os.O_APPEND** tells the OS that writes append to the file, so there is no need to offset to the end of the file. **os.O_RDWR** opens the file for both reads and writes. **os.ModeAppend** only allows writing via appending to the file.
2. Then, we write/append the chunk to the file.
3. We check if the file upload is completed. We do this by checking the range of bytes in the headers.
4. Finally, we upload the file to Google Cloud.

To check that the file is finished downloading, we parse the **Content-Range** header we sent from the browser.
{{< highlight go >}}
func isFileUploadCompleted(c *gin.Context) bool {
	contentRangeHeader := c.Request.Header.Get("Content-Range")
	rangeAndSize := strings.Split(contentRangeHeader, "/")
	rangeParts := strings.Split(rangeAndSize[0], "-")

	rangeMax, e := strconv.Atoi(rangeParts[1])
	if e != nil {
		panic("Could not parse range max from header")
	}

	fileSize, e := strconv.Atoi(rangeAndSize[1])
	if e != nil {
		panic("Could not parse file size from header")
	}

	return fileSize == rangeMax
}
{{< / highlight >}}

The upload to the cloud here is GCP specific, but I will summarize it here if anybody needs it:

1. Create a bucket under **Storage** in [Google Cloud Console](https://console.cloud.google.com). I used Uniform Access Control, but if you need finer access control you can choose the other option. 
<img src="https://www.eventslooped.com/posts/img/chunked-file-upload-typescript-react-go/gcp-storage.png" alt="GCP Storage"/>

2. Create a **service account** to allow access to your resource. This will allow you to generate a json file which stores your credentials. Put this file somewhere in your filesystem and create an environmental variable GOOGLE_APPLICATION_CREDENTIALS to point to this json file. Google provides some instructions [here](https://cloud.google.com/docs/authentication/production#linux-or-macos). 
I provide my own simple instructions for Windows [here](https://www.eventslooped.com/posts/img/gcp-set-service-account-path.png).

3. Use code like this to upload your file.

{{< highlight go >}}
func uploadToGoogle(c *gin.Context, f *os.File) {
	creds, isFound := os.LookupEnv("GOOGLE_APPLICATION_CREDENTIALS")
	if !isFound {
		panic("GCP environment variable is not set")
	} else if creds == "" {
		panic("GCP environment variable is empty")
	}
	const bucketName = "el-my-gallery"

	client, e := storage.NewClient(c)
	if e != nil {
		panic("Error creating client: " + e.Error())
	}
	defer client.Close()

	bucket := client.Bucket(bucketName)
	w := bucket.Object(f.Name()).NewWriter(c)
	defer w.Close()

	fmt.Println()
	fmt.Printf("%v is the upload filename", f.Name())
	fmt.Println()

	f.Seek(0, io.SeekStart)
	if bw, e := io.Copy(w, f); e != nil {
		panic("Error during GCP upload:" + e.Error())
	} else {
		fmt.Printf("%v bytes written to Cloud", bw)
		fmt.Println()
	}
}
{{< / highlight >}}

A few things to mention here. **bucketName** is the name of the bucket you created in step 1. If you forget to close your writer, the upload will still run successfully, but the file will not show up in Google Storage. 
**f.Seek(0, io.SeekStart)** is needed to rewind the reader to the beginning, because we just finished writing to the end of the file. 

### Let's test!

Submit the form with a few files.
<img src="https://www.eventslooped.com/posts/img/chunked-file-upload-typescript-react-go/upload-test1.png" alt="client-side test"/>

Golang console output.
<img src="https://www.eventslooped.com/posts/img/chunked-file-upload-typescript-react-go/upload-test2.png" alt="server-side test"/>

Google Cloud Platform result.
<img src="https://www.eventslooped.com/posts/img/chunked-file-upload-typescript-react-go/upload-test3.png" alt="GCP storage result"/>

Great everything worked :)
<img src="https://aqzodowgen.cloudimg.io/bound/800x600/n/https://www.eventslooped.com/posts/img/chunked-file-upload-typescript-react-go/viktor-talashuk-cabin.jpg" alt="Look what we made!"/>
<a style="background-color:black;color:white;text-decoration:none;padding:4px 6px;font-family:-apple-system, BlinkMacSystemFont, &quot;San Francisco&quot;, &quot;Helvetica Neue&quot;, Helvetica, Ubuntu, Roboto, Noto, &quot;Segoe UI&quot;, Arial, sans-serif;font-size:12px;font-weight:bold;line-height:1.2;display:inline-block;border-radius:3px" href="https://unsplash.com/@viktortalashuk?utm_medium=referral&amp;utm_campaign=photographer-credit&amp;utm_content=creditBadge" target="_blank" rel="noopener noreferrer" title="Download free do whatever you want high-resolution photos from Viktor Talashuk"><span style="display:inline-block;padding:2px 3px"><svg xmlns="http://www.w3.org/2000/svg" style="height:12px;width:auto;position:relative;vertical-align:middle;top:-2px;fill:white" viewBox="0 0 32 32"><title>unsplash-logo</title><path d="M10 9V0h12v9H10zm12 5h10v18H0V14h10v9h12v-9z"></path></svg></span><span style="display:inline-block;padding:2px 3px">Viktor Talashuk</span></a>

### Summary

<p>I think this solution worked out quite well, but there are some things to consider. When using this solution in production, there is a small chance of filename collision; I recommend adding a unique Guid to the name or a userID in multi-tenancy solutions. </p>
<p> Initially I tried to build this app using a more efficient approach, *without saving the file to the filesystem* and appending each chunk to Google Cloud directly as it came in. Unfortunately, I ran into a GCP limitation here. At the time of writing, the maximum allowed update rate of any 1 file is 1 per second. Anything above this rate fails to upload. Therefore, I had to fall back to using the filesystem to put the chunks together. 
