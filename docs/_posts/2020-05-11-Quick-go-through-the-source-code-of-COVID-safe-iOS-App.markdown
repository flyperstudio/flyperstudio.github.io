---
layout: post
title:  "Quick go through the source code of COVID safe iOS App"
date:   2020-05-11 09:18:16 +1100
categories: tech blog
---

Just saw the App open the source code from [Github](https://github.com/AU-COVIDSafe/mobile-ios) and I'm very interested how does this App implement the logic, so I go through the iOS App source code.

The code is very clean and tidy, and I found this project maybe create by [GovTech company](https://www.tech.gov.sg/), cause I found the developer put his name in source code and I found him in LinkedIn, so I guess it should come from this company.

From the mobile client, the App logic is very simple, the first step is to get your personal information and upload to the server, all information is what you type into the App. Then the App will setup a Bluetooth Central and Peripheral meanwhile the server will give each user a unique ID. My ID is a 384 length string, it's not a Base64 string, I guess that string is an encrypted content. When you set up everything, the App will go to the Home page which you will see it when you launch the App.

The App will share the information between devices, in the background, it will find the same services with other Bluetooth devices. The Bluetooth Central will connect with other Bluetooth peripherals and write this device information like this
`{"modelP":"iPhone XR","org":"AU_DTA","msg":"***","v":1}`The other devices will keep this in a local database with this model
![DataModel](https://flyperstudio.github.io/images/4255b5e9-2f28-4bbf-be92-72efbda2df4b.png)

For each communication, there's no data uploaded to the server.

So it just collected all the data from other devices and keep it in local, when you click the "Has a health official contacted you?" button, it will upload all local data to the server. So I guess from the server side it will send push notification to all the devices it kept in the local database.

This App just uses standard iOS API for the Bluetooth staff, as I was developed a Bluetooth App 3 years ago, at that moment, the Bluetooth background connect is not very good. It cannot find the Bluetooth devices immediately when the App was in the background. But in the front ground, it can find it immediately. I have confirmed this with Apple, Apple just said cause the Bluetooth will drain the battery, so it cannot guarantee it will scan and connect every time. So I still have doubt about if this iOS App will collect enough information in the background. Like if you talked with another person about 3 mins, not sure if the App will tracing this.

In the end, I have to say this is a high code quality project and hope this project will help other developers to development Bluetooth App.