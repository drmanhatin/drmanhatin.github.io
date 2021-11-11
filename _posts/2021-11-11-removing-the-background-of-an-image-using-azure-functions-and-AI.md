---
layout: default
title:  "Creating your own image background removal API | Remove.bg clone"
date:   2021-10-26 11:33:27 +0100
categories: azure functions remove.bg clone
description: "For an app I have made, I was looking for an API with which you could remove the background of an image. I found a few but most of them charge around 20 eurocents per image which I found far tooS much. 
So I started looking for an alternative solution."
---

# Creating your own image background removal API

For an app I have made, I was looking for an API with which you could remove the background of an image. I found a few but most of them charge around 20 eurocents per image which I found far too much. 
So I started looking for an alternative solution. Whilst googling I ran into a github repository called ["image-background-remove-tool"](https://github.com/OPHoperHPO/image-background-remove-tool).

This repository contains a python command line application with which you can remove backgrounds from images.


<figure> 
        <img src="/assets/images/kitty.jpg"/>
        <figcaption> Cat with background.</figcaption>
</figure>


<figure> 
        <img src="/assets/images/transpcat.png"/>
        <figcaption> Cat without background.</figcaption>
</figure>



 Perfect, right? Now I want to be able to communicate with this application over HTTP instead of using the command line. I also need to deploy it in a scalable manner - removing backgrounds is pretty CPU intensive work. And that's where Azure Functions are very useful!

<br/><br/>

## Azure Functions
Azure Functions is an Azure serverless solution with which you can easily deploy and run pieces of code. These functions handle scaling, http routing, ssl
encryption and much more. So I, the developer, can focus on writing code instead of thinking about infrastructure and how to make the solution scalable. 
Depending on the configuration, functions can scale (automatically!) from 0 to 200 instances running your code. Scaling to 0 effectively means that when it is not being used, it will not cost you any money. And scaling to 200 instances means you could have 200 instances simulatenously processing your images. That is plenty of room to scale for most applications. 

A downside of the scale to 0 concept is that you may run into cold start issues. A cold start occurs when the function hasn't been used in a while and an instance will need to boot up, load all the dependencies and the machine learning model. Now because this solution is quite large (the model alone is over 150Mb), 
a cold start takes about 30 seconds. 

A way to deal with this problem is by doing "warmup requests". A warmup request is nothing else than a request to the function, which will start the bootup process. In my app I send a warmup request when a user logs in, because I know this Azure function needs to be ready
to handle requests. Then, if the user tries to remove the background of an image, he doesn't have to wait 30 seconds for the process to complete.

<br/><br/>


## How I converted the image-background-remove-tool to an Azure Function
Now the image-background-remove-tool is a console line application. 
If you open the main.py method, that is where the magic happens:
`def process(input_path, output_path, model_name="u2net",
            preprocessing_method_name="bbd-fastrcnn", postprocessing_method_name="rtb-bnb"):`

As you can see this method receives a input path to the image to process. It also expects an output_path, so where to write the processed image.

Now the steps I went through to convert the application into an Azure Function are as follows:
1. Created an Azure Function using the VS Code extension
2. Copied the repository into the Azure Function
3. Edited the main function:


``` 
def main(req: func.HttpRequest) -> func.HttpResponse:

    #generate unique path, to avoid possible collision 
    input_path = get_random_string(8)
    
    #if the body is too small, just use a temporary cat image 
    #(for development & load testing purposes)
    if(req.get_body().__len__()  < 5): 
        input_path = "tempImagePath.jpg"

     
    else:  
        #create a temporary file to store the image which is in the request body
        fp = tempfile.NamedTemporaryFile(delete=False, suffix=".jpg" )
        fp.write(req.get_body())
        fp.close()
        input_path = fp.name
        
    #setup a file to store the image without background
    op = tempfile.NamedTemporaryFile(delete=False, suffix=".png")  
    output_path = op.name
    op.close()
    
    #process it, just as if the app is being called over the command line
    process(input_path, output_path)

    #finally return the image without background
    resp = func.HttpResponse(body=open(output_path, 'rb').read(), 
    mimetype="image/png")

    return resp
```


<br/><br/>

## Performance considerations

1. Writing to the filesystem is probably not the best approach. It would be better to refactor the application to keep the image in memory throughout the process.
2. I have removed all other models included in this application to speed up the startup.
3. I also removed any unnecessary packages from the requirements.txt file.

The algorithm used to replace the "background" pixels with transparent pixels, should be improved. Currently the application can run out of memory if the image is too large.
Same thing applies to the pre/post-processing methods. The algorithms implemented are very cpu/memory intensive and should be replaced/optimized if you want to deploy this to production.





