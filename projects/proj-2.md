---
layout: post
title: 'Project Two'
---

I was very lucky to start my career at an innovative hearing aid technology company. I learned so much and was given so much freedom to explore ideas – it really could not have been a better place to grow as an engineer. This post touches on my first project during my time there, and while I can’t share any code due to some proprietary features involved, I can discuss it.
I was tasked with evaluating whether it was possible to reliably assess if a user had fallen using only an inertial measurement unit (IMU) located in the user’s hearing aids. The IMU consisted of an accelerometer to measure the rate of change of the user’s motion and a gyroscope to measure the orientation of the user.
A secondary objective of this project was to evaluate the contribution that a gyroscope would make to a fall detection system. It was assumed that a gyroscope can indeed aid fall detection by mitigating false positives, e.g., sitting down very quickly. However, gyroscopes are resource intensive devices – they require a constant electrical current to operate – which would significantly degrade battery life.
Finally, I was to investigate the effect that sampling rates played on the prediction.
Of course, this sounds like an ideal case for machine-learning-based approaches. However, remember that hearing aids are very small devices with extremely limited memory – accordingly, the model would have to be extremely small. This means that deep learning approaches were immediately ruled out, as even small DL models require tens of thousands of parameters to store in memory. Not only deep learning was ruled out – useful classical machine learning approaches such as support vector machines also could not be used, as they require each sample to be stored in memory. 
All of these tasks and constraints made for a very interesting problem, and I hope that by the end of this post, you’ll agree as well.

First look at the data

Because there isn’t really any open source data regarding fall detection in hearing aids (surprise!) I began my initial analysis using the IMU data from Simon Fraser Universityhttps://www.sfu.ca/tips/data-sharing.html  and EBU (sorry, no link) in Turkey. However, the datasets were recorded with an IMU sensor located on the forehead, not in the ears. It’s also important to note that the movements are all discrete in time – they are all 10s – 15s files depicting a single movement, and they do not lead into or out of other movements, which isn’t ideal, and is the case for the data that was recorded afterwards using the prototype sensors. In short, these datasets aren’t exactly what I was looking for, but they gave me a great jumping-off point.
The datasets contained fall, activities of daily life (ADLs), and near-fall movements. After looking at the various movement types, it became clear how difficult it would be to differentiate between near-falls and actual falls – think stumbling and catching your balance vs. stumbling and actually falling. 


{% include image.html url="http://www.gratisography.com" image="projects/proj-2/stretch.jpg" %}
