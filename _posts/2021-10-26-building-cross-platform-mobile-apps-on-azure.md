---
layout: default
title:  "Creating a cross platform app on Azure and deploying it to the Play Store."
date:   2021-10-26 11:33:27 +0100
categories: azure static web app pwa cross platform
description: "I think many people are not aware of the possiblities of running mobile apps on Azure. In this blogpost I will demonstrate how easy it is to build mobile apps on Azure and get them into the play store"
---

# Creating a cross platform app on Azure and deploying it to the Play Store. 

A PWA, short for Progressive Web App, is a website which behaves like a classic desktop application. The website (or web app) uses Javascript to facilitate interactions with users. 
Browsing through a classical website means loading entire pages over and over, whereas a PWA makes lightweight API calls to interact with a backend, and after receiving a response the browser then renders the page itself.

A big advantage of using a PWA is that you can use a single codebase and single CI/CD pipeline to deploy the app for multiple platforms. Because the page is rendered on the device (client side rendering) the backend can focus on returning data, and the app can be deployed as static HTML/CSS/Javascript website.

PWA’s are most commonly made using Javascript frameworks such as Vue, React and Angular. In this example I will use Vue, because thats what I’m most familiar with.

We will deploy this Vue PWA using Azure Static Web Apps. 

Static Web Apps are the ideal solution for developers who want to get their website up and running as quickly as possible. Out of the box this service will host your app, provide SSL encryption and create a CI/CD pipeline for you. Oh, and it’s very cheap too! Obviously we will use this service to deploy our PWA :)

If you have all the prerequisites listed below, you can have a working cross platform app working within 15 minutes. 

### The end result
The end result is an app which we can access through the browser but also install as a android app. This app can also be deployed to the Apple app store.


### Prerequisites
If you want to follow this tutorial, you'll need the following:
- Azure Account
- Github Account
- [Google Developer Account](https://play.google.com/)
- [NodeJS + NPM installed](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm)
- [Vue3 & Vue3 CLI installed](https://cli.vuejs.org/)


### Creating the project
Vue supports a CLI with which you can create new projects. The command is ‘‘vue create $nameofapp”. Then, manually select features and tick the PWA support box.

<figure> 
        <img src="/assets/images/vue3cli.png"/>
        <figcaption>Vue CLI</figcaption>
</figure>

A new project is created in your current directory. We won’t go too much in depth on what all the various files are responsible for, but lets name a few important ones:


<figure> 
        <img src="/assets/images/vuedir.png"/>
        <figcaption>Vue Directory Structure</figcaption>
</figure>


#### Package.json
This file describes your project, what depencies are required to run it and what steps are performed to build and serve the project. 

#### SRC directory
This directory contains the files which make up the PWA. At the root of this directory you can find App.vue, a Vue component file. Vue components are typically built up of three parts: 
- a HTML template, describing what is in the page
- some javascript code, where the logic on how to interact with the component is defined
- a CSS stylesheet, which describes how the page looks like

Now commit the project to a github repository, so we can get to the next step, which is deploying the project.

### Deploying the project
We will deploy the project using Azure Static Web Apps. The easiest way to do this is through the portal. Link your Github account and select the repository you have uploaded the Vue3 app to.  

<figure> 
        <img src="/assets/images/staticdeployment.png"/>
        <figcaption>Static Web App Creation</figcaption>
</figure>

Azure Static Web Apps will create a Github Actions pipeline for you, which automatically deploys new versions of the app when you commit to this github repository.


 Under Build Details, select the VueJS build preset.

<figure> 
        <img src="/assets/images/vue3builddetails.png"/>
        <figcaption>Static Web App Build Details</figcaption>
</figure>

The app location field should match the directory in which you have the package.json file located. In my case, this file is in the root of the repository. 

Finish creating the webapp and browse to the github repository. You will notice that Azure has created a Github Action, which will be used to build and deploy your Vue3 PWA to Azure.

<figure> 
        <img src="/assets/images/githubaction.png"/>
        <figcaption>Our Github Action pipeline</figcaption>
</figure>


Open the Static Web App you just created in the Azure Portal, and click the “Browse” button. 

<figure> 
        <img src="/assets/images/vue3pwabrowse.png"/>
        <figcaption>Static Web App Browse</figcaption>
</figure>


If all went well, you should see your Vue3 PWA, now accessible over the internet. If it is still “waiting for content”, your CI/CD pipeline may not be finished yet. Alternatively, check that you have provided the right repository, app location and that you have actually committed all the files to the repository.

### Creating an Android App from the static web app
Microsoft has built an awesome tool, [PwaBuilder](https://www.pwabuilder.com), to convert web apps into phone apps. Browse to the site and enter the URL of your Static Web App. Click next and navigate to the Android package.

<figure> 
        <img src="/assets/images/vue3androidbutton.png"/>
        <figcaption>PWABuilder Android Button</figcaption>
</figure>

Download the generated package and unzip it. 

<figure> 
        <img src="/assets/images/vue3package.png"/>
        <figcaption>Android Package</figcaption>
</figure>

The package contains 6 files. We will use two of them to upload our app to the play store. The assetlinks file is used to prove that the creator of the website has also created a phone app with which you can access it. This file prevents people from making phone apps from sites which they do not run. This file must always be accessible in the same path: https://YOURSITEURLHERE.com/.well-known/assetlinks.json

<figure> 
        <img src="/assets/images/vue3dir.png"/>
        <figcaption>Vue 3 directory structure</figcaption>
</figure>

We will create a .well-known folder in the public folder of the Vue project and copy the Assetlinks.json file in there. 

The .aab file contains our actual Android app. The last step is to upload this file to the Play Store.


### Creating a Play Store app
This is probably the most time consuming step of the project, as you may need to create a developer account on the [Google Play Console](https://play.google.com/)

After finishing this process, access the Play Console and create a new app.

<figure> 
        <img src="/assets/images/vue3createapp.png"/>
        <figcaption>Android console app creation</figcaption>
</figure>

Fill in the fields and create the app.

Now the last step is to create an internal release of this app, so we can allow specific users to download and test the app. From the side menu, open the “Internal Testing” page, create a release, and upload the .aab file (located in the Zip file downloaded from pwabuilder.com). Give the release a name and finish the release. If you are prompted with an error regarding COVID-19 testing apps, browse to the “App Content” page from the sidebar and tell Google that this is not a COVID19 testing app.

<figure> 
        <img src="/assets/images/vue3internaltest.png"/>
        <figcaption>Vue 3 directory structure</figcaption>
</figure>

After creating the internal release, navigate back to the “Internal Testing” page and add some testers (maybe yourself?). 

Then copy the link and open it on your mobile device. You will be redirected to the Play Store page where you can download your app. Now download and run the app!

### Hey, there’s still a browser bar in my app!
Most likely you will still see a browser bar in your app. So by now I think you will fully understand what is happening. This is just a website pretending to be a phone app? Well, kind of. Biggest difference is that this app can work offline aswell, by using [Service Workers](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Offline_Service_workers). But thats a bit too in depth for this tutorial.

Now lets make sure users don’t see this ugly browser bar in the app. 

<figure> 
        <img src="/assets/images/vue3browserbar.png"/>
        <figcaption>Vue 3 directory structure</figcaption>
</figure>

Go back to the Play Console and browse to the App Integrity tab. 
Copy the SHA-256 certificate fingerprint. Now go back to your Vue project and open the assetlinks.json file. Overwrite the existing SHA-256 fingerprint with this new one. Now, commit all changes (git add *, git commit -m “change fingerprint”, git push). The Github Action will be triggered and the change will be deployed to your app in a few minutes. After a few minutes, * CLOSE * the app on your phone and open it again. Voila! No more browser bar. 

Now what we have setup is a very simple basis on which we can develop a cross platform capable mobile app.
- This app can run in the browser
- Or can be deployed as an Android app
- and can also be deployed as an iOS app.
- A single CI/CD pipeline can update the app for all platforms at once. 

A good PWA is (nearly) indistinguishable from a native app. If you don’t need access to special native API’s or if you don’t need to absolute best CPU performance,  consider using a PWA instead of building seperate native apps. Furthermore,modern browsers support an increasing amount of device API’s. Below you can see an example of a phone app I made (also using Vue 3), which includes a photo editor and a camera. All made in Javascript! All cross-platform compatible!


<figure> 
        <img src="/assets/images/vue3coolfeatures.png"/>
        <figcaption>Displaying modern PWA capabilities</figcaption>
</figure>



{% include share-buttons.html %}










