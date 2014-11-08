TL;DR: SoundCloud changed their terms of service and politely asked me to remove my Android app from the Play Store. I’m open sourcing it today and you can get it [from GitHub](https://github.com/passy/scdl).

[![unnamed.webp.png](https://d23f6h5jpj26xu.cloudfront.net/tiw0erpnallmla_small.png)](http://img.svbtle.com/tiw0erpnallmla.png)

It was an amazing journey and I’m very grateful for the services SoundCloud provided for free. More than **71 releases**, **2,571,289 installs**, **11,257,885 downloaded songs**, **5,498,590 swipes** on the tutorial screen and **22,233 reviews** later, **Downloader for SoundCloud is no more.**

## The Idea

SoundCloud always had a great feature in its desktop web version. You can directly download any song to your hard drive. That is, if the artist is so kind to tick a checkbox during the upload flow. Depending on the genre you’re into this is more or less common, but especially for longer mixtapes it’s still quite prevalent.

[![0.png](https://d23f6h5jpj26xu.cloudfront.net/mf98lmjwwcs0w_small.png)](http://img.svbtle.com/mf98lmjwwcs0w.png)

Unfortunately neither the mobile web version nor the official native clients allow this. Data storage and file management is difficult, especially within the UX constraints on mobile devices, so I wasn’t surprised when I noticed that this feature was missing.

I remember searching for an app that would fill this gap in late April 2012. After entering "soundcloud d" into the searchbox of the Play Store, I saw an auto-suggest not unlike todays.

[![0 2.png](https://d23f6h5jpj26xu.cloudfront.net/4oir9lbofrgvqq_small.png)](http://img.svbtle.com/4oir9lbofrgvqq.png)

To my surprise, however, after tapping on the first suggestion, the result page was empty. There was in fact not a single app providing a way to store SoundCloud songs on your device.

## Best Market Validation Ever

Given that there was clearly a big demand for an app like this, I sat down for a weekend and built a prototype. The idea was simple. By hooking into Android’s Intent infrastructure I wouldn’t have to reimplement song search but directly add a "link" to my app in SoundCloud’s share dialog.

[![0 3.png](https://d23f6h5jpj26xu.cloudfront.net/8emai4sultogew_small.png)](http://img.svbtle.com/8emai4sultogew.png)

After a quick look at SoundCloud’s open API I was pleasantly surprised that it included all the information I required, including a direct download link. Even better, the terms of use explicitly mentioned the download of songs, which was restricted to those with an enabled *downloadable* flag. After finding out about Android’s integrated [DownloadManager](https://developer.android.com/reference/android/app/DownloadManager.html) it was only gluing the pieces together.

After less than two days, I had a first working version of the app. As you can’t expect users to read the Play Store (or Android Market back then) description to figure out how to use the app, I also added a quick screenshot-based tutorial explaining the steps from the SoundCloud app to playing the locally stored file from the device.

## To The Moon

Even with the strong indicator of people’s interest I had no expectations of the app taking off. After all, I was just scratching my own itch. So without spending more time on polish, I put the app on the store. I didn’t charge money for it, which seemed obvious to me given that I was leveraging SoundCloud’s free API. I did, however, add a single AdMob banner to the bottom of the download screen.

Checking the stats on the next day, I couldn’t believe my eyes. More than 100 downloads in less than half a day, exclusively from organic searches through the Play Store. To me, this was a success. The numbers kept going up and the reviews were generally favorable, except for the occasional 1-star reviews complaining about being unable to download songs the artist didn’t grant the permission to.

The second time I thought my eyes were deceiving me was when I read a review requesting the ability to pay for the removal of the single ad in the app. I always wanted to play around with In-App Purchases, so I wouldn’t have cared if the only user of this feature for this one guy from Romania leaving the review. But my expectations were exceeded once again. On the day I launched the new version including the IAP I didn’t get one but three purchases of the "Remove Ads" In-App product for 1.49€, $1.99 or ₤1.49, respectively. I’m still unclear about whether people saw this as a way to say thank you or whether the banner really bothered them that much. Either way, over time the IAP made up for about ⅓ or my revenue, with the remaining ⅔ coming from ad impressions. The downloads stabilized over the years between 7000 and 7500 per day.

[![image_3.png](https://d23f6h5jpj26xu.cloudfront.net/erbnzkep9mloa_small.png)](http://img.svbtle.com/erbnzkep9mloa.png)

## Trouble In Paradise

During the 2 ½ years of the app’s life on the Play Store it was taken down three times. The first time was a particularly unpleasant experience. I woke up one morning looking at a notice from the Play Store telling me that my app was suspended for copyright and trademark infringement. I figured that this was probably because of the name "SoundCloud Downloader" and the use of the SoundCloud Logo as part of my app icon. The SoundCloud terms of service allowed the use of their resources to my understanding for products building on top of their official API. But as there was no way to ask for further details or appeal to the suspension, I submitted a new app with the name “Downloader for SoundCloud” and using a custom icon instead, hoping for the best. At this point, the app had more than 40 000 installs 200 reviews that I would lose. Also, there were several alternative apps at this point I now had to catch up with in the rankings.

Luckily, I managed to make my way to the top again and got the pole position in "Users who installed SoundCloud also installed X" suggestion within a couple of weeks. A few months later, another suspension. This time I with a bit more information, though. The request came directly from SoundCloud and the email included contact information for a community manager at SoundCloud that I should get in touch with. It turned out that my app was collateral damage in an attempt to suspend all apps that allowed unauthorized downloads. Unfortunately, there were quite a few apps with the exact same name and I was caught in the crossfire. On the upside, I now had an official blessing from SoundCloud and a lot less competition in the market.

The same process happened two more times and always meant a few weeks without new installs or revenue and the fear of getting banned forever.

[![Screenshot from 2014-11-08 17:19:17.png](https://d23f6h5jpj26xu.cloudfront.net/bry0cqtdya5iyq_small.png)](http://img.svbtle.com/bry0cqtdya5iyq.png)

## All good things …

On the 15th of September this year, I received an email about my app, coming directly from SoundCloud’s Trust and Safety team this time. They had revised their terms of use and now included two new clauses which directly affected me:

* "Your app must not offer offline access to audio User Content."

* "Your app must not be specifically designed to cache any User Content."

I knew that this day would come and frankly expected it way sooner. There was no point in arguing, since this decision was clearly entirely at SoundCloud’s discretion. They politely asked me to remove my application from the store or alternatively revoke my API key. Naturally, I complied with their request and suspended the app on the same day.

It was an amazing experience having your code run in so many hands all over the world and I was incredibly lucky finding just the right niche at the right time. It also served as a strong reminder how dependent you are on the benevolence of the app store owners. I was lucky in that I always treated this app as an experiment that could be over at any point, but if your entire business and livelihood depends on it, the idea that it could all be gone in the blink of an eye is not exactly comforting. This is yet another reason why I hope that the web will ultimately win and the need for closed and tightly controlled ecosystems like the app stores are no longer needed.

## The Downloader is Dead, Long Live the Downloader

Even though the app itself isn’t very useful anymore, I hope the code could still be, so I’m open sourcing the app including the entire Git history:

[https://github.com/passy/scdl](https://github.com/passy/scdl)

*With thanks to Stephen Sawchuk for his corrections and suggestions.*
