---
layout: post
title: 'Fall detection in hearing aids'
---
**Tech used: Scikit-learn, accelerometer, gyroscope**

I was very lucky to start my career at an innovative hearing aid technology company. I learned so much and was given so much freedom to explore ideas – it really could not have been a better place to grow as an engineer. This post touches on my first project during my time there, and while I can’t share any code due to some proprietary features involved, I can discuss it.

#### Requirements and constraints

I was tasked with evaluating whether it was possible to reliably assess if a user had fallen using only an inertial measurement unit (IMU) located in the user’s hearing aids. The IMU consisted of an accelerometer to measure the rate of change of the user’s motion and a gyroscope to measure the orientation of the user.
A secondary objective of this project was to evaluate the contribution that a gyroscope would make to a fall detection system. It was assumed that a gyroscope can indeed aid fall detection by mitigating false positives, e.g., sitting down very quickly. However, gyroscopes are resource intensive devices – they require a constant electrical current to operate – which would significantly degrade battery life. Finally, I was to investigate the effect that sampling rates played on the prediction.

Of course, this sounds like an ideal case for machine-learning-based approaches. However, remember that hearing aids are very small devices with *extremely* limited memory – accordingly, the model would have to be *extremely* small. This means that deep learning approaches were immediately ruled out, as even small DL models require tens of thousands of parameters to store in memory. Not only deep learning was ruled out – useful classical machine learning approaches such as support vector machines also could not be used, as they require each sample to be stored in memory. All of these tasks and constraints made for a very interesting problem, and I hope that by the end of this post, you’ll agree as well.

#### First look at the data

Because there isn’t really any open source data regarding fall detection in hearing aids (*shocking*, I know), I began my initial analysis using the IMU data from Simon Fraser University and Bilkent University (sorry, no link) in Turkey. However, the datasets were recorded with an IMU sensor located on the forehead, not in the ears. It’s also important to note that the movements are all discrete in time – they are all 10s – 15s files depicting a single movement, and they do not lead into or out of other movements, which isn’t ideal, and is the case for the data that was recorded afterwards using the prototype sensors. In short, these datasets aren’t exactly what I was looking for, but they gave me a great jumping-off point.

The datasets contained fall, activities of daily life (ADLs), and near-fall movements. After looking at the various movement types, it became clear how difficult it would be to differentiate between near-falls and actual falls – think stumbling and catching your balance vs. stumbling and actually falling. 

{% include image.html image="projects/proj-2/fall-vs-nearfall.png" %}
{% include image.html image="projects/proj-2/fall-vs-nearfall_mag.png" %}

As usual, before we can do any real analysis however, I had to clean the data. Both datasets contained a few missing rows, so nothing major there. Just get rid of those rows. The file structures, however, were quite different from one another. The movements needed to be in the correct folders, and, since the datasets had different sampling rates, they needed to be resampled. This cleaning up, resampling, and reorganization process was done with various functions developed using MATLAB. In the end, I had a small dataset of approximately 3,700 movements.

#### Feature extraction

After much thinking and analyzing various movements, I came up with a handful of features that I though might be useful. To confirm the usefulness of the features, I created histograms of each feature for each movement type and compared them against one another to judge the separability and uniqueness of the features.

{% include image.html image="projects/proj-2/feats1.png" %}
{% include image.html image="projects/proj-2/feats2.png" %}
{% include image.html image="projects/proj-2/feats3.png" %}

Features can be considered useful or representative of different movement types if their histograms are fairly separate from one another. In other words, if they have little overlap with other movement types. For my purposes, it was not important that near-falls and ADLs have too much overlap. The importance lay with a high degree of separability between the fall movement from the others. The features shown above are some of the more interesting features that I found. After having found my features, it was time to make a first prototype model to test how difficult the classification process would be. For this simple prototype, I chose to go with a decision tree for its ease of use. Initial tests were quite promising, showing approximately 97% accuracy using both the accelerometer and gyroscope.

{% include image.html image="projects/proj-2/initial-conmatrix.png" %}

The initial tests are nice to see, but they aren’t really indicative of how the system would perform in a real-time setting. The model is seeing the entire movement in a 10s – 15s vector, whereas in a real-time system, the model would be looking at much shorter intervals – intervals that are far too short to fit the entire movement. 

#### Choosing the the ML models

Obviously the decision tree is far too simplistic for use in a critical application. Small changes in data, such as a "unique" fall, could dramatically change how the tree classifies a movement. I liked the idea of using a tree-based method here, however, so I went with a gradient booster algorithm. The gradient booster is similar to a random forest, in that it uses many weak classifiers - the shallow decision trees - to classify movements. However, the gradient booster weighs the more difficult to classify movements more heavily than those that are easy to classify, and the next tree is trained using the modified tree/weights. In this way, the many weak learners become strong learners. But I didn't want to look at just one possibility - that would be rather boring, right? What about using something like a Gaussian Mixture Model (GMM)? 

{% include image.html image="projects/proj-2/densities.png" %}

The image above demonstrates the core idea of a GMM. Distributions of falls should look different from nonfalls, and therefore, should be able to be differentiated by looking at the distribution. Using a GMM would be also be optimal for memory-saving reasons - they only need to store the mean and standard deviations of a movement type to make classifications. And assuming falls are different enough from ADLs and near-falls, their distributions should be different enough to differentiate movements based on these statistical properties. So in the end, I investigated two different models - the gradient booster classifier and the GMM.

#### Developing the real-time model

Having finally obtained the primary dataset (in which I was lucky enough to both lead portions of and participate in others), I was ready to begin developing a model that should make classifications in real-time. This data differed significantly from the open-source dataset I had been previously using in that the hearing aids contained sensors that were directly in the ears of the subject rather than placed on the head itself, and in that each of the new recordings formed a continuous stream of several movements which lasted approximately 7 minutes as opposed to individual discrete movements lasting only seconds long. In total, there were approximately 24 hours of data. Below is a high-level overview of how this real-time model was developed.

{% include image.html image="projects/proj-2/overview.png" %}

Real-time systems are typically developed to read data in overlapping chunks or frames. However, the data was not labeled on a per-frame basis – rather each movement consisted of a start time and end time, and it was the individual samples between these start and end times that were labeled. This meant that each recording needed to be broken into frames and relabeled according to the original labels. Don’t forget that each recording needed to be resampled to find the ideal sampling frequency!

{% include image.html image="projects/proj-2/resampling-labels.png" %}

But this too, held another challenge. For fall movements, subjects lay often simulated a loss of consciousness, which was also labeled as part of a fall. Obviously, a subject lying down or resting should not be labeled as a fall. However, that simulated unconsciousness is crucial to determining if a fall had occurred. Therefore, a function was written that would intelligently separate each movement into frames and relabel the frames based on the amount of energy in the signal and by slicing any excess “unconsciousness time.” After the signal was broken into frames, the features were extracted, and the label added to each feature vector. These feature vectors were then concatenated to one another to form a final feature matrix – one for each sampling frequency.

{% include image.html image="projects/proj-2/feature-extraction.png" %}

After each recording had been broken into frames, resampled, and labeled, the recordings were imported into a Python environment to begin the training phase. For this task, I used Scikit-Learn, a machine learning library with tons of great resources from preprocessing to fitting a model and making predictions. Both the GMM and gradient boosting models were subjected to a 5-fold exhaustive grid search in an effort to identify the optimum parameters for fall detection. These models were fitted with the optimum parameters, saved to the hard drive, and then imported back into MATLAB for the final prediction. 

{% include image.html image="projects/proj-2/truth-v-predictions.png" %}

The image above shows the ground truth signal produced by the labelling function, and on the bottom are the predicted falls shown in black, and the probability that a movement was a fall shown in pink. I have removed the y-axis from the bottom image in order to enhance the readability of the probability line. These two lines, the ground truth and the prediction line, are then compared sample-by-sample to calculate the accuracy. This is shown in the confusion matrix on top, where we see that no fall was missed. However, we see 1160 samples misclassified. This is in part due to the false positive shown on the right, but also because, if you look closely, the prediction frames are slightly larger than the labeled ground truth frames. This is because a fall is occurring in multiple frames, and I had not yet found an elegant way to handle this in my algorithm. Nevertheless, the fall is actually occurring, but when checking sample-by-sample, the larger prediction frame skews the result. Therefore, this could be looked at as a worst-case scenario.

#### Evaluating the model

{% include image.html image="projects/proj-2/gb-results.png" %}

Pictured here are the balanced accuracies for selected sampling frequencies when using the gradient boosting algorithm, as a function of the frame size, which is some integer value of the y-axis multiplied with the sampling frequency, and the step size, which is the frame size divided by an integer value of the x-axis. Higher accuracies are shown as a dark blue, while poor accuracies are shown as a lighter blue. I selected only three of the investigated sampling frequencies to give you an idea of how a low, middling, and higher sampling frequency might impact the prediction accuracy. At 16 Hz, using frame sizes of 4 to 6 seconds, we can see that we generally achieve a higher accuracy when considering only the acceleration data. The highest accuracy here is 95.78% compared to a high of 93.5% when considering both sources. Moving to 64 Hz, we again see that the values for a frame size of 4 seconds with a step size of 1/3 to 1/4 of the frame size is superior when considering only the accelerometer data, with a 98.53% accuracy compared to 97.67% and 97.0% compared to 96.8% respectively. And finally, at 200 Hz, this trend still holds true – accuracy is about 1.1% to 1.2% higher when looking only at the accelerometer. 

{% include image.html image="projects/proj-2/gmm-results.png" %}

And here we have the same evaluation for the GMM model. The trend continues to hold true for this model as well. Across all sampling frequencies, we generally can observe a higher prediction accuracy when considering only the accelerometer data.

Let’s take a look at comparing the performance of the two models against one another. Below, we look at one activity block, in which there are 6 falls. Both models have the same sampling frequency, frame size, and step size, and both models are considering only the acceleration data. Looking at the confusion matrices, we can see that neither model has missed any of the 6 falls, which is great. However, there is a stark contrast in the number of false positives. The gradient boosting algorithm, in general, has a much higher precision, as evidenced here by the single false positive. The GMM on the other hand, is quite a bit more sensitive, showing several false positives. This is generally true for all sampling frequencies and is likely the reason behind the GMM Model displaying so few false negatives.

{% include image.html image="projects/proj-2/fp-acc.png" %}

Looking now at the effect that the gyroscope has on the prediction accuracy, we can see right away that the gyroscope has increased the number of false positives for the gradient boosting algorithm, from one to three. This is apparent by the reduced accuracy shown in the confusion matrix. However, the gyroscope data has actually increased the prediction accuracy of the GMM model, despite actually increasing the activities that prompted a false positive. Here we see an additional false positive that did not previously exist when looking only at the accelerometer data. This additional misclassification is only offset by the removal of a few false positives.

{% include image.html image="projects/proj-2/fp-acc-gyro.png" %}

#### Summary of results – or, TL;DR

We can generally observe that the prediction accuracy when only considering the acceleration data is superior to the prediction accuracy when considering both sensors. There may be a few more niche movements that could improve the detection of a fall via the gyroscope, but when looking at the current results, it doesn’t seem to warrant an inclusion of the gyroscope. The highest accuracy attained in this study was 98.53% via the gradient boosting algorithm at 64 Hz. The GMM model seems to prefer smaller sampling frequencies with larger step sizes, while the gradient boosting model prefers smaller to medium sampling frequencies with smaller step sizes. 
And that’s all she wrote. This was a lot of fun to work on, and it’s even more special because of the nature of this kind of product can have a meaningful impact on the user’s life – the potential medical applications could be genuinely lifesaving, if not at least life-improving. I hope you found this write-up interesting, and maybe even learned something. 😉
