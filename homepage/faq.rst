==========================
Frequently Asked Questions
==========================

This FAQ wasn't created by myself. Its content is almost exclusively from the `Imaging Knowledge Base - FAQ <http://mindhive.mit.edu/node/46>`_ from the `Gabrieli Lab at MIT <http://gablab.mit.edu/>`_. It is my thanks to them, for generating such an exhaustive FAQ.

**Index of FAQ**

* `Basic Statistical Modeling`_
* `Connectivity`_
* `Contrasts`_
* `Coregistration`_
* `Design`_
* `HRF`_
* `Jitter`_
* `MATLAB Basics`_
* `Mental Chronometry`_
* `Normalization`_
* `P threshold`_
* `Percent Signal Change`_
* `Physiology and fMRI`_
* `ROI`_
* `Random and Fixed Effects`_
* `Realignment`_
* `Scanning`_
* `Segmentation`_
* `Slice Timing`_
* `Smoothing`_
* `SPM in a Nutshell`_
* `Temporal Filtering`_

.. note::

    To help me to extend this FAQ, please feel free to leave a comment at the bottom to recommend additions and to clear misapprehension.


Basic Statistical Modeling
===================

1. What is "estimating a model?" How do the various programs do it?
*********************
Once you've performed all the spatial preprocessing you like on your functional data, you're ready to test your hypotheses about the data. Most standard analysis pathways in fMRI proceed in a hypothesis-driven fashion based on the general linear model (GLM), in which the researcher sets up a model of what she believes the brain may be doing in response to the variations in some parameter of the experiment, and then tests the truth of that hypothesis. (This contrasts with non-model-driven approaches like principal components analysis (PCA). Estimating your model is the core statistical step of this process: the researcher describes some model of brain activity, and the program calculates a giant multiple regression of some kind to find out the extent to which that model correctly accounts for the real data, at every voxel in the brain.

Different programs have different methods of setting up a design matrix, but they all share certain elements: the user describes a set of different experimental conditions (or effects), and describes, for each of them, start times and end times (onset times and offset times, or durations). Experiments can have massively varying designs, from the simplest on-off block design to multi-condition randomly-timed event-related design. The basic hypothesis is that some voxels in the brain had their intensity values covary, to a statistically significant degree, with some combination of the experimental conditions and parameters. The design matrix consists of a matrix with a row for each timepoint in the experiment (each functional image), and a column for each modeled experimental effect.

Usually, the user will then modify the design matrix to make it a more accurate model of what brain activity might be. Oftentimes, a constant term is added to the matrix, to account for the mean value of the session; sometimes linear or polynomial drifts are added to the matrix as well. Sometimes the columns of the matrix are convolved with some model of the hemodynamic response function, to reflect the blurring in signal the HRF applies to neural activity. (Another option is to simply separate the various timepoints for the response to a given condition into different columns, estimating each separately - a finite impulse response (FIR) model that effectively deconvolves the contribution of the HRF.) (See `HRF`_ for more info.)

Once the design matrix is set up, the program uses the methods of GLM theory - essentially multiple regression - to calculate how accurately the model described by the design matrix accounts for the real data. The standard GLM equation is Y = BX + E, where Y is the time-varying intensities from one voxel, X is the design matrix, E is an error term, and B is the "parameters" or "beta weights" - a vector of values, one for each experimental condition, that tells the researcher how big the effect of the corresponding condition was in explaining the values at that voxel. If condition A's beta weight is significantly greater than condition B's beta weight at a given voxel, the hypothesis that A had a greater effect than B at that voxel is confirmed. Generally, programs create some voxel-by-voxel image of the beta weights - a beta image or parameter image.

Once the parameters are estimated, the program has both a measure of effect size and of error in the model for each voxel. Generally, the program then normalizes each effect size by the error to calculate some measure of statistical significance for effects - a contrast image (see `Contrasts`_ for more info). 

2. When should global mean scaling be used? What does it do?
*********************
Nutshell answer - global mean scaling should be used for PET, but not for fMRI.

Longer answer: One problem in neuroimaging experiments is that you're generally trying to pick out some signal from a noisy timeseries at every voxel. One form that noise can take is a global shift in intensities across the whole brain, which can be caused by scanner thermal noise, subject movement, physiological effects, etc. One way to get rid of a whole bunch of those noise sources at once, then, would be to look for timepoints where every voxel in the brain shows the same sudden shift and infer that that's a change in global response, not a regional change, and therefore not of interest to you. A simple way of doing that is just by finding the global mean of every timepoint - a global mean timeseries - and dividing every voxel's timeseries by the global mean timeseries.

The obvious problem with this is that removing the effect of the global mean from your model means you also remove signal that covaries with the global mean. In PET, this wasn't a big deal - the changes in the global mean could be on a different order of magnitude from task-related changes, and so regional activations weren't likely to bias the global mean particularly. In fMRI, though, it's a problem. Global mean intensity shifts are often about the same size, or at least comparable, to the size of task-induced activations. So large activations can seriously bias the global mean calculation, such that the global mean will go significantly up with large activations. Removing the effect of the global mean will then remove those large activations as well.

Using global scaling has been tested fairly extensively now in fMRI, and it almost always seems to negatively affect the sensitivity of the analysis. Generally, it's a bad idea.

3. What is autocorrelation correction? Should I do it?
*********************
The GLM approach suggested above and used for many types of experiments (not just neuroimaging) has a problem when applied to fMRI. It assumes that each observation is independent of the next, and any noise present at one timepoint is uncorrelated from the noise at the next timepoint. On that assumption, calculating the degrees of freedom in the data is easy - it's just the number of rows in the design matrix (number of TRs) minus the number of columns (number of effects), which makes calculating statistical significance for any beta value easy as well.

The trouble with this assumption is that it's wrong for fMRI. The large bulk of the noise present in the fMRI signal is low-frequency noise, which is highly correlated from one timepoint to the next. From a spectral analysis point of view, the power spectrum of the noise isn't flat by a long shot - it's highly skewed to the low frequency. In other words, there is a high degree of autocorrelation in fMRI data - the value at each time point is significantly explained by the value at the timepoints before or after. This is a problem for estimating statistical significance, because it means that our naive calculation of degrees of freedom is wrong - there are fewer degrees of freedom in real life than if every timepoint were independent, because of the high level of correlation between time points. Timepoints don't vary completely freely - they are explained by the previous timepoints. So our effective degrees of freedom is smaller than our earlier guess - but in order to calculate how significant any beta value is, we need to know how much smaller. How can we do that?

Friston and Worsley made an early attempt at. They argued that one way to account for the unknown autocorrelation was to essentially wash it out by applying their own, known, autocorrelation - temporally smoothing (or low-pass filtering) the data. If you extend the GLM framework to incorporate a known autocorrelation function and correctly calculate effective degrees of freedom for temporally smoothed data. This approach is sometimes called "coloring" the data - since uncorrelated noise is called "white" noise, this smoothing essentially "colors" the noise by rendering it less white. The idea is that after coloring, you know what color you've imposed, and so you can figure out exactly how to account for the color.

SPM99 (and earlier) offer two forms of accounting for the autocorrelation - low-pass filtering and autocorrelation estimation (AR(1) model). The autocorrelation estimation corresponds more with pre-whitening, although it's implemented badly in SPM99 and probably shouldn't be used. In practice, however, low-pass filtering seems to be a failure. Tests of real data have repeatedly shown that temporal smoothing of the data seems to hurt analysis sensitivity more than it helps, and harm false-positive rates more than it helps. The bias in fMRI noise is simply so significant that it can't be swamped without accounting for it. In real life, the proper theoretical approach seems to be pre-whitening, and low-pass filtering has been removed from SPM2 and continues to not be available in other major packages. (See `Temporal Filtering`_ for more info.)

4. What is pre-whitening? How does it help?
*********************
The other approach to dealing with autocorrelation in the fMRI noise power spectrum, instead of 'coloring' the noise, is to 'whiten' it. If the GLM assumes white noise, the argument runs, let's make the noise we really have into white noise. This is generally how correlated noise is dealt with in the GLM literature, and it can be shown whitening the noise gives the most unbiased parameter estimates possible. The way to do this is simply by running a regreession on your data to find the extent of the autocorrelation. If you can figure out how much each timepoint's value is biased by the one before it, you can remove the effect of that previous timepoint, and that way only leave the 'white' part of the noise.

In theory, this can be very tricky, because one doesn't actually know how many previous timepoints are influencing the current timepoint's value. Essentially, one is trying to model the noise, without having precise estimates of where the noise is coming from. In practice, however, enough work has been done on figuring out the sources of fMRI noise to have a fairly good model of what it looks like, and an AR(1) + w model, where each noise timepoint is some white noise plus a scaling of the noise timepoint before it, seems to be a good fit (it's also described as a 1/f model). This procedure essentially estimates the level of autocorrelation (or 'color') in the noise, and removes it from the timeseries ('whitening' the noise).

Theoretically, it should work well, but as its adoption is relatively new to the field, few rigorous tests of the effectiveness of pre-whitening have been done. 

5. How does parametric modulation work? When would I use it?
*********************
As described above, there are all kind of modifications the researcher can make to her design matrix once she's described the basics of when her conditions are happening. One important one is parametric modulation, which can be used in a case where an experimental condition is not just ON or OFF, but can happen at a variety of levels during the experiment. An example might be an n-back memory task, where on each trial the subject is asked to remember what letter happened n trials before, where n is varied from trial to trial. One hypothesis the research might have is that activity in the brain varies as a function of n - remembering back 3 trials is harder than remembering 1, so you might expect activity on a 3-back trial to be higher than on a 1-back. In this case, a parametric modulation of the design matrix would be perfect.

Generally, a parametric modulation is useful if you have some numerical value for each trial that you'd like to model. This contrasts with having a numerical value to model at each timepoint, which would be a time for a user-specified regressor. In the parametric case, the user specifies onset times for the condition, and then specifies a parameter value for each trial in the condition - if there are 10 n-back trials, the user specifies 10 parameter values. The design matrix then modulates the activity in that column for each trial by some function of the parameter - linear, exponential, polynomial, etc. - set by the user. If the hypothesis is correct, that modulated column will fit the activity significantly better than an unmodulated effect.

6. What's the best way to include reaction times in my model?
*********************
If you have events for which participants' response times vary widely (or even a little), your model will be improved by accounting for this variation (rather than assuming all events take identical time, as in the normal model). A common way of including reaction times is to use a parametric modulator, with the reaction time for each trial included as the parameter. In the most common way of doing this, the height of the HRF will be thus modulated by the reaction time. Grinband et al. (HBM06) showed this method actually doesn't work as well as a different kind of parametric regression - in which each event is modeled as an epoch (i.e., a boxcar) of variable duration, convolved with a standard HRF.

In other words, rather than assuming that neural events all take the same time, and the HRF they're convolved by varies in height with reaction time (not very plausible, or, it turns out, efficient), the best way is to assume the underlying neural events vary in reaction time, and convolve those boxcars (rather than "stick functions") with the same HRF.

In either case, as with most parametric modulation, the regressor including reaction time effects can be separate from the "trial regressor" that models the reaction-time-invariant effect of the trial. This corresponds to having one column in the design matrix for the condition itself (which doesn't have any reaction time effects) and a second, parametrically modulated one, which includes reaction times. If your goal is merely to get the best model possible, these don't need to be separated (only the second of the two, which includes RTs, could go in the model), but this will not allow you to separate the effect of "just being in the trial" from neural activations that vary with reaction time. To separate those effects, you need separate design matrix columns to model them. That choice depends on how interested you are in the reaction-time effect itself.

7. What kinds of user-specified regressors might I use? How do I include them?
*********************
Another modification you can make to the design matrix is simply to add columns or effects that don't correspond to some condition you want convolved with an HRF. A user-specified regressor is just some vector of numbers, one for each timepoint/functional image, that you'd like to include in the model because you believe it has some effect. If you have a numerical value for each timepoint (TR/functional image) that you'd like to model, a user-specified regressor is the way to go. This contrasts with the case of having a numerical value for each trial you'd like to model, in which case you'd use a parametric modulation.

An example of a user-specified regressor might be if you have continuous self-reports of positive affect from each subject, and you'd like to see where there are voxels in the brain whose activity co-varied with that affect. You could include the positive affect regressor in your model and have a beta value estimated separately for it. Depending on what your hypothesis is about that effect, you may want to lag its values to account for the hemodynamic delay.

The user-specified regressor is a powerful tool for many types of modifications to the design matrix, but note that in many obvious cases in which you might want to separate out the contribution of a given effect of no interest - things like movement parameters, physiological variation, low-frequency confounds, etc. - programs may already have ways to deal with those things built in, in a more efficient fashion. At the very least, in any case when you include a user-specified regressor than you plan to simply ignore, you should try to ensure it doesn't covary significantly with your task and hence remove task-induced signal.


Connectivity
===================

1. What is functional connectivity? What is effective connectivity?
*********************
The concept of "brain connectivity" is, as Horwitz points out, rather a tricky one to define. Ideally, you'd like to be able to measure the spatial (and temporal) path that information follows, from one point to another, millisecond by millisecond, and neuron to neuron (or at least region to region), in a directed fashion, such that you could say, "Ah, yes, activation starts in the visual cortex, moves to V2, gets shuttled from there to these other three visual areas and parietal cortex, and from parietal there to this other bit." Then you'd know something about what was being calculated and what calculations were being done where (and when). But, of course, you can't do that (yet). In fact, in general, most neuroscience recording methods, be they single-cell recording or fMRI or anything else, deal with isolated units of analysis - single cells or single voxels. You can't, in general, really well measure one neuron's connection to another in a living, behaving animal, much less noninvasively in a person.

What you can do is sample several sites at once and try and see how the patterns of activity you get are connected to each other. An obvious pattern to look for would be if two sites/voxels have intensities that are highly correlated. If the timeseries from one voxel looks exactly like the timeseries from another voxel, it might be a good bet they're doing similar things. If they're right next to each other, you call it a cluster; if they're far away from each - say, in visual cortex and PFC - you might guess they're connected to each other somehow.

Trouble is, of course, you run into the old adage that correlation doesn't imply causation. High correlation between remote sampling sites might imply some direct connection, or it might imply some third site driving their joint activation, or it might imply them jointly driving some third site. And even if they are connected, it's difficult to tell the direction of the connection, even if there is a "direction" to it.

Hence the two different terms used to describe connectivity in neuroimaging, a split introduced by Friston in 1993. Functional connectivity is the correlation concept - it's a descriptive concept, simply defined as the temporal correlation between remote timeseries or samples or what have you. Finding functional connectivity essentially reduces, as Lee et. al point out, to finding whether activity in two regions share any mutual information or not. Effective connectivity, by contrast, is the causation concept. It's defined as "the influence one neural system exerts over another either directly or indirectly." It doesn't imply a direct physical connection - simply a causative influence. It's a concept meant to support explanation and inference, more than just description, and it requires some account of causative direction or why there isn't any. It's also a lot trickier to figure out, generally, than functional connectivity. You'll hear both terms tossed around a fair amount, but remember: functional is simply correlation, whereas effective requires some causation somewhere.

2. How do I measure connectivity in the brain?
*********************
Good question. Almost every method for functional neuroimaging has ways to measure connectivity, and almost all of them boil down to the same concept: measuring the connection between timeseries at different points in the brain. In other words, almost every method out there measures functional connectivity, rather than directly measuring effective connectivity. Whether you're doing EEG and correlating timeseries from different electrodes, or using the fanciest dynamic causal modeling mathematical strategy with fast-TR fMRI, you're restricted generally to the data you can measure, which are samples from voxels that are treated independently. You can rule out some possible directions of influence by rules like temporal precedence (if a spike in one area precedes one from another area, the latter area can't have caused the earlier spike), but in general, most connectivity measures work on this simple foundation: Sample timeseries of activity from many different areas (voxels, electrodes, etc.) and then mathematically derive some measure of the mutual information between selected timeseries.

There are a couple obvious exceptions to this foundation. Measuring anatomical connectivity is a different type of procedure, and it's not clear how much influence anatomical connectivity (as we can measure it) and functional connecitivity have with each other - or should have with each other in analysis. Lee et. al has an intriguing discussion on this point (and many others). At some level, of course, if we're interested in finding out whether information is flowing from one neuron to another, it's useful to know if they're directly connected or not. But we're a long way off from those sorts of measures on a large scale, and it's not clear that coarser measures are all that useful in learning about functional connectivity. Anatomical connectivity on its own can be incredibly interesting, though, which is why diffusion tensor imaging (DTI) is becoming increasingly popular as an imaging modality. The idea of DTI is that it can extract a measure of directionality of the white-matter tracts in a given voxel, giving you a picture of where white matter is pointing in the brain. This can be used to infer which areas are strongly connected to each other and which less so. And, of course, many older techniques for measuring connectivity in animals - staining, tracing, etc. - are still widely used.

The other big exception to the rule of measuring correlation is techniques that can directly measure causality by disrupting some part of the system. If you can knock out part of the system and cause a part hypothesized to depend on it to fail, while knocking out the latter part doesn't affect the former, you can start to make some inferences about directionality of influence. In living humans, the latest way to do this is with transcranial magnetic stimulation (TMS), which seems to offer some ways to disrupt selected cortical areas temporarily, reversibly and on command. Although the technique is relatively new, it holds high promise as an additional tool in the connectivity toolbox. Other methods for disruption - cortical cooling, induced lesions in animals, even lesion case studies in humans - can provide valuable information on this front as well.

3. What are the different methods to analyze connectivity in fMRI? How do they differ from each other?
*********************
In a field burdened with a heavy load of meaningless acronyms and technical jargon, connectivity analyses stand out as a particular offender. There are what seems like a dizzying array of ways to model connectivity in fMRI, each with various acronyms and fancy-sounding concepts and a great number of equations underlying it. The important thing to remember in all of them is that the data input is essentially the same: it's just timeseries data. And the underlying computations are all essentially doing the same thing - looking for patterns in the data that are similar between regions. Some methods literally attempt to do the same thing, but for the most part, the different methods proliferate because they examine slightly different aspects of connectivity. So the important things to think about when faced with interpreting or performing any connectivity analyses are goals: what is the point of this analysis? What does its output measure? What are the alternate possible explanations that this analysis has ruled out, or failed to rule out?

There's a broad distinction you can make in connectivity analyses between model-driven and non-model-driven analyses. Non-model-driven analyses are those which don't "know" anything about the details of your experiment - some of them are called "blind" algorithms because they're searching for patterns in your data without knowing what the structure of your experiment was. These types of analyses can be used to get at activation in general, but they're probably more widely used in connectivity analyses. Principal components analysis (PCA) and independent component analysis (ICA) are non-model-driven types of analyses. I won't talk much about them yet here, because I don't know much about them right now. Anyone else out there, feel free to contribute some info...

The popular forms of model-driven analysis are those embedded into the popular neuroimaging programs, and SPM2 has recently added a couple to the field which are getting wide use. Psychophysiological interactions (PPIs) have been used in SPM and other programs for a while, but SPM2 has automated this analysis to make it a lot easier to perform. They start with a seed ROI and look for other regions that have high changes in connection strength to the seed as the experiment proceeds. Dynamic causal modeling (DCM) is Karl Friston's latest addition to the modeling tradition, and it's a much more all-encompassing form of connectivity analysis; he claims it subsumes all earlier forms of analysis as well as the standard general linear model activation analysis. DCM starts with a set of ROIs and attempts to determine the influence of each on the other and the experiment's influence on the connectino strengths. BrainVoyager is soon to release a connectivity package based on the concept of autoregressive modeling and Granger causality - a way of ruling out some directions of causality. This isn't out yet, due to a patent dispute, so I don't know much about it. Structural equation modeling (SEM) is used when you have a set of ROIs you'd like to investigate, but aren't sure what the links between them may be; it's a way of searching among the possible graphs that connect your ROIs and ruling out some connections while including others. Which of these you decide to use will decide on your experimental goals and what you want this analysis to show, exactly.

4. What is a psychophysiological interaction (PPI) analysis? How do I do it? Why would I want to?
*********************
A PPI analysis starts with an ROI and a design matrix. It's a way of searching among all other voxels in the brain (outside the seed ROI) for regions that are highly connected to that seed. One of the most straightforward ways of doing connectivity analyses would be to start with one ROI and simply measure the correlation of all other voxels in the brain to that voxel's timeseries, looking for high correlation values. As Friston and other pointed out a while ago, though, it's not quite as interesting if the correlation between two regions is totally static across the experiment - or if it's driven by the fact that they're both totally non-active during rest conditions, say. What might be more interesting is if the connection strength between a voxel and your seed ROI varied with the experiment - i.e., there was a much tighter connection during condition A between these regions than there was during condition B. That may tell you something about how connectivity influences your actual task (and vice versa).

PPIs are relatively simple to perform; you extract the timeseries from a seed voxel or ROI and convolve it with a vector representing a contrast in your design matrix (say, A vs. B). You then put this new PPI regressor into a general linear model analysis, along with the timeseries itself and the vector representing your contrast; you'll use those to soak up the variance from the main effects, which you'll ignore in favor of the PPI interaction term. When you estimate the parameters of this new GLM, the voxels where the PPI regressor has a very high parameter are those who showed a signficant change in connectivity with your experimental manipulation.

This is do-able in SPM99, or indeed any program; SPM2 makes it more automated, and adds some mathematical wrinkles, like deconvolving the HRF from your PPI regressor so as to look for interactions at the deconvolved (and hopefully neural) level, rather than at the HRF level.

PPIs are good to do if you have one ROI of interest and want to see what's connected with it. They're tricky to interpret, and they can take a really long time to re-estimate if you have several ROIs to explore and many subjects.

5. What is structural equation modeling (SEM)? How do I do it? Why would I want to?
*********************
Structural equation modeling analyses begin with a set of ROIs and nothing else. The idea in SEM is to try and estimate connection strengths between those ROIs that make up the best possible model of connection between them. The connection strengths are correlational (not directional), but represent the straightforward degree of correlation between the timeseries of those regions. This strategy (and variants of it) also fall under the title "path analysis," although that's a broader term that can describe analyses of non-timeseries data. SEM procedures can vary, but they're all kind of like the GLM: they search through the space of possible connection strengths until they find the set of connection strengths that best fits the data.

The measure of "best fit" is an important choice in SEM, and there's not wide agreement on the measure you should use, except a common suggestion that you use more than one and combine their results. Other bells and whistles on SEM analyses can include bootstrapping the data (see `P threshold`_ for information on permutation tests and bootstrapping) to get a confidence interval on how good the model could possibly be (Bullmore et. al (2000), NeuroImage 11, describe this strategy).

SEM isn't built in to any of the major neuroimaging programs that I know of, but several statistics program support it (as it's used in other social sciences besides neuroscience).

SEM is good to do when you have a set of ROIs - either functional or anatomical - and you're interested in knowing how strong the connections are between them (or whether connections between a particular pair exist at all) across the whole experiment (or part of it). It's a pretty straightforward style of analysis, but because of that, it doesn't take into account of lot of details of fMRI - temporal variations in the connection strengths, for example.

6. What is Granger causality? How does it relate to brain connectivity?
*********************
Granger causality is a concept imported from economics, where it was developed to do timeseries modeling of economic data (weird that that's the kind of data economists would want to look at - economic data, you know. Strange guys, those economists). It's an attempt to impose some directionality on connections between timeseries, or at least rule out some directions, by leveraging the rule of temporal precedence. The core of the Granger causality idea is that events can't cause events that already happened - so if a particular pattern happens in one timeseries, and then happens later in another timeseries, the latter one can't have caused the former one. Granger causation is a very limited form of causality, because it doesn't rule out the possibility that some third factor has induced the change in both of the timeseries, or any of the other problems common to ascribing causality to correlation data, but it's a start in the direction of blocking off certain directions of arrows.

The most explicit use of Granger causality has been in the connectivity package being developed for BrainVoyager, detailed below in Goebel et. al. The package is based also on the use of vector autoregressive modeling, which I couldn't begin to explain in detail but I gather is kind of like dynamic causal modeling or something like that. Unfortunately, the package has been held up in patent disputes, so it's not clear when we'll get to evaluate it up front. I'm not aware of other packages or programs currently using those methods to evaluate connectivity.

7. What is Dynamic Causal Modeling (DCM)? How do I do it? Why would I want to?
*********************
Well, this is another one that I'm undoubtedly going to botch the explanation for. But I'll take a very limited stab at it. If you're interested in this analysis, I highly recommend reading Friston et. al's paper on it at Connectivity.

DCM analyses are highly model-driven. You start with a set of ROIs and a guess at how they're connected with each other. That guess can be "fully connected," with every ROI attached to every other, or you can eliminate some connections off the bat. DCM then takes as its input your design matrix and the timeseries from those regions, and attempts a sort of hyper-advanced general linear model estimation. Instead of a general linear model, though, DCM explicitly considers some nonlinear aspects to the experiment: specifically, the connections between your ROIs and how they might change with the experimental manipulation. It goes through a huge set of Bayesian estimations and deconvolutions and every other fancy thing you can think of, and what you get on the way out is a big set of parameters. That set will include: HRFs for each of your regions, "resting" connection strengths between each of your regions, beta weights describing how the experiment affected each of your regions (just like regular beta weights), and "connection beta weights," indicating how the experimental manipulation affected your connection strengths. It'll also spit out some estimation of the statistical significance of each of these.

Friston et. al are hyped on this analysis; they believe that all the other analyses out there (SEM, PPI, etc.) are all special cases of DCM. Even the standard general linear model analysis of activation is a special case, they say, where you're assuming there are no connections between ROIs, and your ROIs are your voxels. A few papers have been put out thus far - Mechelli et. al (below) is one - using DCM in big analyses, with fairly promising results.

DCM is built into SPM2, and requires you to have SPM2 results to use it. It's not available yet for any other neuroimaging program.

DCM is great if you've got a set of ROIs, a hypothesis about how they might work, and you're particularly interested in how some areas or conditions might influence the connections between some other areas. Mechelli et. al is a good example of this - they looked at whether differences in visual activations due to categories of stimuli were mediated from the bottom up or from the top down. It's also kind of insanely complicated right now, and clearly in a sort of feeling-out phase in the community. Results may be difficult to interpret. But it's definitely the cutting edge of fMRI connectivity research for model-driven analyses right now.

8. How do I measure connectivity across a group?
*********************
Almost all of these methods measure correlations between timeseries, and so they're only appropriate to do at the individual level. The best way to run a group analysis is in the standard hierarchical fashion - take the output of the individual analysis and toss it into a group analysis. The output from all of them won't be the same - for PPIs, for example, you'll get an activation image, which works in SPM for a standard group-level analysis, whereas for SEM you'll get a set of connection weights, which you can then run a standard statistical test on in SPSS - but the hierarchical approach should work fine in general.


Contrasts
===================

1. What's the difference between a T- and an F-contrast? When should I use each one?
*********************
Simply put, a T-contrast tests a single linear constraint on your model - something like "The effect size (parameter weight) for condition A is greater than that for condition B." T-contrasts can involve more than two parameters, but they can only ever test a single sort of proposition. So a T-contrast can test "The sum of parameters A and B is greater than that for parameters C and D," but not any sort of AND-ing or OR-ing of propositions.

An F-contrast, by contrast (ha!), is used to test whether any of several linear constraints is true. An F-contrast can be thought of as an OR statement containing several T-contrasts, such that if any of the T-contrasts that make it up are true, the F-contrast is true. So you could specify an F-contrast like "parameter A is different than B; parameter C is different than D; parameter E is different than F," and if any of those linear contrasts were significant, the F-contrast would be significant. The utility of the F-contrast is highest when you're just trying to detect areas with any sort of activation, and you don't have a clear idea as to the shape of the response. They were designed to be used with something like a Fourier basis set model, where you want to know if any combination of your cosine basis functions is significantly correlated with the brain activation. Testing that set with a T-contrast wouldn't be correct; it would tell you whether the sum of those basis functions' parameters was significant, which isn't what you'd want. Testing individually whether any of those parameters is significant, though, tells you something.

The disadvantage of the F-test is that it doesn't tell you anything about which parameters are driving the effect - that is, which of the linear constraints might be individually significant. It also doesn't tell you what the direction of the effect; parameter A might be different than parameter B, but you don't know which one is greater. This isn't a problem if you're using a basis set where different parameters don't have much individual physiological meaning (such as a Fourier set), but oftentimes F-tests are followed up with t-tests to further isolate which parameters are driving the effect and what direction the effect is in.

The Ward, Veltman & Hutton, and Friston papers on Contrasts both describe the F-test and how it's used in pretty clear fashion, with specific examples.

2. What's a conjunction analysis? How do I do one?
*********************
An F-test allows you to OR together several linear constraints, but what if you want to AND them together? That is, what if you want to test if all of a set of several linear constraints are satisfied? For that, you need a conjunction analysis. There are several ways to perform them - see the Price & Friston paper on Contrasts and those below it - but SPM provides a built-in way that is a good example. (Details of how to use SPM to do one are in the Veltman & Hutton paper there). The idea is to find the intersection of all the sets of voxels that satisfy a given linear constraint in the set, a simple mathematical operation in itself. The tricky part is to figure out what threshold level to use on each individual linear constraint to give the conjunction (or intersection) an appropriate p-threshold. SPM makes the choice that the p-thresholds on each individual constraint simply multiply together, so a conjunction of two constraints that you wanted to threshold at 0.001 would mean thresholding each individual constraint at the square root of 0.001. The resulting field of t-statistics is called a "minimum T-field" - effectively you're thresholding the smallest T-statistic among the linear constraints at each voxel - and SPM allows corrected p-thresholds to applied as well as uncorrected. These analyses are also available for F-constrasts, to AND together several OR statements.

One problem that some critics of this approach have highlighted is that it means at a voxel called "active" in the conjunction, any individual constraint on it may hardly be significant at all. If you want to see the conjunction of contrasts A and B, you'd prefer not to see 'common activations' that have p-values far above a reasonable threshold when looked at in each individual contrast. Price & Friston have argued that the individual constraints don't matter much in conjunctions, but some people still prefer not to use the minimum T-field approach for this reason. In this case, you can conjoin constraints together simply by intersecting their thresholded statistic maps (with some care taken to make sure the contrasts are orthogonalized (see below)), which can be done algebraically.

3. What does 'orthogonalizing' my contrast mean?
*********************
If you're testing a conjunction, one worry you might have is the the contrasts that make it up don't have independent distributions - that they are testing, to some degree, the same effect - and thus the calculation of how significant the conjunction of will be biased. If you use SPM to make a conjunction analysis through the contrast manager, it will attempt to avoid this problem by orthogonalizing your contrasts - essentially, rendering them independent of one another. The computation involved is complicated - not just simply checking whether the contrast vectors are linearly independent, although it's derived from that - but it can be thought of as follows:

Starting with the second contrast, check it against the first for independence; if the two are not orthogonal, remove all the effects of the first one from the second, creating a new, fully orthogonal contrast. Then check the third one against the second and the first, the fourth against the first three, and so on. SPM thus successively orthogonalizes the contrasts such that the conjunction is tested for correctly. See the help docs for spm_getSPM.m for more details.

4. How do I do a multisubject conjunction analysis?
*********************
Friston et. al is a good paper to check out for this. They describe some ways of thinking about the SPM style of conjunction analysis, which is normally a fixed-effects and hence only single-subject analysis, that allow its extension to a population-level inference. It's not clear that all the assumptions in that paper are true, and so it's on a little shaky ground.

However, it's certainly possible at an algebraic level to intersect thresholded t-maps from several subjects, just as easily as it is from several constraints. So it may make sense to try the simple intersection method, using somewhat loosened thresholds on the individual level. I'm not super sure on all the math behind this, so you might want to talk to Sue Gabrieli about this sort of thing...

5. What does the 'effects of interest' contrast image in SPM tell you?
*********************
Not an awful lot of interest, as it turns out. It's an image automatically created as the first contrast in an SPM analysis, and it consists of a giant F-contrast that tests to see whether any parameter corresponding to any condition is different from zero. In other words, if any of the columns of your design matrix (that aren't the block-effect columns) differ significantly from zero, either positively or negatively, at any voxel, that voxel will show up as significant in this F-image. Needless to say, it's not a very intepretable image for anyone who isn't using a very simple implicit-baseline design matrix. So generally, don't worry about it.

6. How is the intercept in the GLM represented in the analysis?
*********************
Every neuroimaging program accounts for the "whole-brain mean" somehow in its statistics, by which I mean whatever part of the signal that does not vary at all with time. That time-invariant point can be represented in the design matrix explicitly as a column of all ones, and SPM automatically includes a column like that for each session in a given design matrix. (AFNI and BrainVoyager don't explicitly show this column in the design matrix, but they include it in their model in the same fashion.) During the model estimation, a parameter is fit at each voxel to this whole-experiment mean, as any other column of the design matrix, and its value represents the mean signal value around which the signal oscillates. This is the 'intercept' of the analysis - the starting value from which experimental manipulations cause deviations. This number is automatically saved at each voxel in SPM ( in the beta images corresponding to the block effect columns) and can be saved in AFNI or BrainVoyager if desired.

7. How do I make contrasts for a deconvolution analysis? What sort of contrasts should I report?
*********************
Generally, deconvolution analyses of the sort implemented by AFNI's 3dDeconvolve work on a finite impulse response (FIR) model, in which each peristimulus timepoint for each condition out to a threshold timepoint is represented by a separate column in the design matrix. In this case, a given 'condition' (or trial type) is represented in the matrix not by one column but by several. The readout of the parameter values across those peristimulus timepoints then gives you a nice peristimulus timecourse, but how do you evaluate that timecourse within the GLM statistical framework? There are a couple of ways; in general, the Ward is the best reference to describe them.

A couple obvious ones, though. First, an F-contrast containing a single constraint for each column of a given condition will test the 'omnibus' hypothesis for that condition - the hypothesis that there's some parameter significantly different from zero somewhere in the peristimulus timecourse, or more, simply, the hypothesis that there was some brain signal correlated to the task at some point following the task onset. This test won't tell you what sort of activity it was, but it will point out areas that had some sort of activity of some kind going on.

Secondly, a variety of different T-contrasts could be used to test various hypotheses about the timecourse. You might be interested in testing between two conditions at the same timepoint that you think might be the peak of the HRF. You might be interested in whether a single condition's HRF rose more sharply or fell more sharply (in which case a T-contrast within that timecourse could be used). You might use some sort of a summing T-contrast to compare the 'area below the curve' in two different conditions.

There's not wide consensus about exactly what sorts of statistics count as 'significant' activation at this point in the literature - the difference between an HRF that rises sharply, spikes high, then falls back down to baseline quickly from an HRF that rises slowly, peaks only a little above baseline, but stays above baseline for a long time, isn't real clear at this point. No one is sure what such a difference represents exactly. This means, though, that there are a wealth of differences between timecourses that one could potentially explore. Almost any hypothesis can be made interesting with the right explanation, and fortunately almost any hypothesis can be tested in the GLM with the tools of T-tests, F-tests and conjunctions of constraints.


Coregistration
===================

1. What is coregistration?
*********************
Remember realignment? It's just like that. It's a way of correcting for motion between images. But coregistration focuses on correcting for motion between your anatomical scans and your functional scans. The slightly trickier thing about that is that your anatomical scans might be T2-weighted, while your functionals are T1-weighted. Or maybe you have a Spoiled Grass anatomical and PET functional images. The intensity-based motion correction algorithms kind of choke on those. So coregistration aims for the same result as realignment - lining up two neuroimages - but uses different strategies to get there.

2. What are the different ways to coregister images?
*********************
These days, there are a few, but one de facto standard, which is coregistration by mutual information (MI). In some ways, coregistration is always the same problem - it's just like realignment, but can be between different modalities (PET, MRI, CAT, etc.), and usually you can be slower at it. The problem boils down to finding some function that measures the difference between your two images and then minimizing (or maximizing) it. Minimization/maximization is pretty standard these days; the question is what sort of cost function you use.

In realignment, we just used the sum of the squared difference in intensity between the images, measured voxel-by-voxel. The trouble with these scheme in coregistration is that your images might be different modalities, and hence a tissue type that's very dark in one (say, ventricle in PET) might be very bright in another (say, proton-density-weight MRI). In that case, trying to minimize the intensity differences between the images will give you a horrible registration.

So there are a couple strategies. SPM99 and older used templates within each modality that were already coregistered by hand with each other; that way, you could just realign your images to their modality-specific templates and automatically put them in register. These days, though, almost all automated coregistration schemes (including SPM2) use MI or some derivative of it.

3. What is mutual information, exactly?
*********************
In a nutshell: If two variables A and B are completely independent, their joint probability Pab (the probability that A comes up a at the same time that B comes up b) is just the product of their respective separate probabilities Pa and Pb; so Pab=Pa*Pb, if A and B are independent. On the other hand, if A and B are completely dependent - that is, knowing what value A takes tells you exactly what value B will have - then their joint probability is exactly the same as their respective separate probabilities; so Pab = Pa = Pb, if A and B are completely dependent. If A and B are dependent a little bit, but not entirely, their joint probability will be somewhere in between there - knowing what value A has tells you a little bit about B, so you can make a good informed guess at what B will be, but not know it exactly.

Mutual information is a way of measuring to what extent A and B are dependent on each other. Essentially, if you can estimate the true joint probability Pab for all a and b, and you know the individual probability distributions Pa and Pb, you can measure how far away the probability distribution Pab is from Pa*Pb, with a Kullback-Leibler statistic that measures the distance between curves. If Pab is much different from Pa*Pb, then you know A and B are dependent to some degree.

Alternatively, you can frame MI in terms of uncertainty; MI is the reduction in uncertainty about B you get by looking at A. If you're much more certain about A after looking at B, then A and B have high MI; they're quite dependent. If you don't know anything more about A after looking at B, then they have low MI and are pretty independent.

4. So how does mutual information help coregistration?
*********************
MI coregistration methods work by considering the intensity in one image to be A and the intensity in the other image to be B. The algorithm computes the MI between those two variables - finds the mutual information between the intensity in one image and the intensity in the other - and then attempts to maximize it.

The idea is that, instead of squared-intensity-difference methods which assume that a bright voxel in one image must be bright in another, you let the images themselves tell you how they're related. If by looking at a bright voxel in one image, though, tells you almost infallibly that the corresponding voxel in the other image is dark, then the images have very high MI, and they're probably close to registered. You can leave unspecified the relationship between intensities in the two modalities, and let the algorithm figure out how they're related - it automatically maximizes whatever relationship they have. This makes MI ideal for coregistering a wide variety of medical images.

5. How are coregistration and segmentation related?
*********************
Fischl et. al make the point that the two processes operate on different sides of the same coin - each one can solve the other. With a perfect coregistration algorithm, you could be maximally confident that you could line up a huge number of brains and create a perfect probability atlas - allowing you the best possible prior probabilities with which to do your segmentation. In order to do a good segmentation, then, you need a good coregistration. But if you had a perfect segmentation, you could vastly improve your coregistration algorithm, because you could coregister each tissue type separately and greatly improve the sharpness of the edges of your image, which increases mutual information.

Fortunately, MI thus far appears to do a pretty good job with coregistration even in unsegmented images, breaking us out of a chicken-and-egg loop. But future research on each of these processes will probably include, to a greater and greater extent, the other process as well. Check out `Segmentation`_ for more info on segmentation.


Design
===================

The major tradeoff in planning your experimental design, from a statistical standpoint, is the fundamental one between efficiency and power. In the context of fMRI, power is the conventional statistical concept of how likely it is that your experiment will correctly reject the null hypothesis of no activation; it might be thought of at how good your experiment is at detecting any sort of activation at all. Efficiency, by contrast, is the ability to accurately estimate the shape of the hemodynamic response to stimuli - the variability that your average detectable hemodynamic response has. This is clearly important if you're interested in looking at response timecourse or HRF shape, but also important in finding activations at all - if variability in your modeled response is high, it's more difficult to distinguish one condition from another.

The tradeoff between power and efficiency is, unfortunately, inescapable (shown by Liu et. al) - you can't be optimal at both. Things that increase power include block design experiments, very high numbers of trials per condition, and increased numbers of subjects. Things that increase efficiency include designs with randomized inter-trial intervals (also called inter-stimulus intervals or ISIs) and analyzing your design with an event-related model (whether the design was blocked or not). Semi-random designs can give you a good dollop of both power and efficiency, at the cost of increased experimental length. Where you fall in designing your experiment will depend on what measures you're interested in looking at - but within the given constraints of a particular number of subjects, a reasonable experimental length, and a guess at how big an effect you'll have, there are good steps you can take to optimize your design.

Experimental design is heavily mixed in with setting your scanning parameters, and jittering your trial sequence, so be sure to check out the other design-related pages:

* `Scanning`_
* `Jitter`_
* `Physiology and fMRI`_


1. What are some pros and cons of block designs?
*********************
Pros: High power, lower number of trials and subjects needed. Cons: Low efficiency, high predictability (which may be a problem for certain tasks from a psychological perspective).

2. What are some pros and cons of event-related designs?
*********************
Pros: High efficiency even at lower trial numbers, can have randomized stimulus types and ISIs. Cons: Low power, more trials/subjects needed, more difficult to design - efficiency advantages require jitter (see `Jitter`_) or randomized ISIs.

3. What's the difference between long and rapid event-related designs? What's good and bad about each?
*********************
Long event-related designs have long ISIs - usually long enough to allow the theoretical HRF to return to baseline (i.e., 20 or 30 sec). Rapid event-related designs have short ISIs, on the order of a few seconds. Long event-related designs have generally fallen out of favor in the last few years, as proper randomization of ISI allows rapid designs to have much greater efficiency and greater power than long. Until the very late 1990s, it wasn't entirely clear that rapid event-related designs would work from a physiological perspective - that the HRFs for different trials would add roughly linearly. Since those assumptions have been (more or less) vetted, the only advantage offered by long event-related designs is that they're much more straightforward to analyze, and that rarely outweighs the tremendous advantages in efficiency and power offered by the increased trial numbers of rapid designs.

4. What purpose would a mixed (block and event-related) design serve? Under what circumstances would I want to use one? How do I best design it?
*********************
Mixed designs, which can include both block and event-related periods, or semi-random designs which have blocks of relatively higher and lower odds of getting a particular trial type, can give you good power and efficiency, but at the cost of longer experiments (i.e., more trials). They're more predictable than fully randomized experiments, which may be a problem for certain tasks. AFNI, SPM and Tom Liu's toolbox all have good utilities to design semi-random stimulus trains.

5. How long should a block be?
*********************
From a purely theoretical standpoint, as described by Liu and others, blocks should be as long as possible in order to maximize power. The power advantage of a block comes from summing the HRFs into as large a response as possible, and so the highest-power experiment would be a one-block design - all the trials of condition in a row, followed by all the trials of the next condition. The noise profile of fMRI, however, means that such designs are terribly impractical - at least one and probably two alternations are needed to effectively differentiate noise like low-frequency drifts from the signal of your response. So from a theoretical standpoint, Liu recommends a two- or three-block design (with two conditions, two blocks: on/off/on/off, with three conditions, two blocks: A/B/C/A/B/C, etc.). With few conditions, this can mean blocks can be quite long.

In practice, real fMRI noise means that two or three-block designs may have blocks that are too long to be optimal. Skudlarksi et. al, using real fMRI noise and simulated signal, recommend about 18 seconds for complex cognitive tasks where the response time (and time of initial hemodynamic response onset) is somewhat uncertain (on the order of a couple seconds). For simple sensory or motor tasks with less uncertainty in that response, shorter blocks (12 seconds or so) may be appropriate. Of course, you should always take into account the psychological load of your blocks; with especially long blocks, the qualitative experience may change due to fatigue or other factors, which would influence your results.

Bledowski et al. (2006), using empirically derived estimates of the HRF, mathematically derive a 7-sec-on, 7-sec-off block pattern as being optimal for maximizing BOLD response, suggesting it's a bit like a "swing" - pushing for the first half, then letting go, maximizes your amplitude.

6. How many trials should one block have?
*********************
As many as you can fit in to that time. The more trials the better.

7. How many trials per condition are enough?
*********************
In terms of power, you can't have too many (probably). The power benefits of increasing number of trials per condition continue increasing until at least 100 or 150 trials per condition (see Desmond & Glover and Huettel & McCarthy). In terms of efficiency, 25 or more is probably enough to get a good estimate of your HRF shape.

8. How can I estimate the power of my study before I run it?
*********************
Several of the papers below have detailed mathematical models for trying to figure that sort of thing out; if you can make an educated guess at how large (in % signal change) your effect might be from the literature, Desmond & Glover can give you a decent range of estimation.

9. What's the deal with jitter? What does it mean? Should I be doing it?
*********************
Jitter probably deserves its own FAQ, so check out `Jitter`_ for more info about it...

10. Do I have to have the same number of trials in all my conditions?
*********************
This question comes up especially for subsequent memory analyses, or things like it, where subjects might have only remembered a fraction of the trials they've looked at, but have forgotten a whole lot. If you're trying to compare remembered against forgotten in that case, is that okay? Depends on exactly the ratio. First and foremost, if a given condition has too few trials in general, you'll lose a lot of ability to detect activation in it - as above, if you don't have at least 25 trials in a condition in an event-related study (over the whole experiment), you're probably starting to get on thin ice in terms of drawing inferences between conditions. But the ratio of trial numbers between conditions can also have an influence. Generally, neuroimaging programs assume that the different columns of the design matrix you're comparing have equal variance, and a vast difference in numbers between them will invalidate that assumption. In practice, this probably isn't a huge concern until you're dealing with ratios of 5 or 10 to 1. If you have 35 trials in one condition and 100 in another - it's not ideal, but you probably won't be too fouled up. If you have 30 in one and 300 in another... it's probably cause for some concern.

11. How many subjects should I run? How many do I need?
*********************
Short answer: 20-25 subjects is a good rule of thumb. Long answer: Obviously this is affected to some degree by situations like funding, etc. But from a statistical perspective, this question boils down to what the levels of noise in fMRI are, or a power analysis: how many subjects should you have in order to detect a reasonably-sized effect a reasonable amount of the time? Using moderate estimates of effect sizes (0.5%) and estimating within- and between-subject noise from real data, Desmond & Glover (2002) calculated that 20-25 subjects were needed for 80% power (i.e., chance of detecting a real effect) with about the loosest reasonable whole-brain p-threshold. Smaller effect sizes might require more subjects for the same power, and looser p-thresholds (i.e., for an a priori anatomical hypothesis) might require fewer subjects. But in general, the 20-25 subject barrier is a pretty good rule of thumb. You aren't ever hurt by more subjects than that (although very large sample sizes can start tongues wagging about how small your effect size is, and you don't want to get into a fight about size - we're adults, after all). But unless you're very sure your effect size is much bigger than average, having fewer than 20-25 subjects means you're likely to be missing real effects. Check out Desmond & Glover for detailed analysis.


HRF
===================

1. What is the 'canonical' HRF?
*********************
The very simplest design matrix for a given experiment would simply represent the presence of a given condition with 1's and its absence with 0's in a that condition's column. That matrix would model a signal that was instantly present at its peak level at the onset of a condition and instantly offset back to baseline with the offset of a trial. It's possible this may be a good model of neuronal activity, but since fMRI measures BOLD signal rather than neuronal activity directly, it's clearly not that good a model of the hemodynamic response that BOLD signal represents.

If we assume the hemodynamic response system is linear, linear systems theory tells us that if we can figure out the hemodynamic response to an instantaneous impulse stimulus, we can treat our real paradigm as the conglomeration of many instantaneous stimuli of various kind and the hemodynamic response should sum linearly. Tests over the last ten years suggest that the brain does, in fact, largely behave this way, so long as stimuli are spaced more than a couple hundred milliseconds apart. So the canonical HRF is a mathematical model of that impulse response function. It's a function that describes what the BOLD signal would theoretically be in response to an instantaneous impulse. Once your design matrix is described, your analysis software convolves it with a canonical HRF, so that your matrix now represents a gradual rise in activity and gradual offset that lines up with a 'typical' HRF.

Most of the common neuroimaging programs use similar canonical HRFs - a mixture of gamma functions, originally described by Boynton's group. This function has been found to be a roughly good model of hemodynamic response - at least in visual cortex - in most subjects. It models a gradual rise to peak (about 6 seconds), long return to baseline (another 10 seconds or so) and slight undershoot (around 10-15 seconds), the whole thing lasting around 30 seconds or so.

2. When should you use the canonical in your model? When should you use different response functions? (HRF, HRF w/ derivatives, etc.) What's the difference?
*********************
Generally, the canonical HRF is a decent fit to the true HRF for many normal subjects in many cortical and subcortical regions. If your analysis is intended primarily to test a hypothesis about neural activity, looking only for size and place of activation, and you believe your subjects to be reasonably normal, the canonical makes good sense. If you're looking to find out more about your activation than simply where it is and how big it is, though, you'll need to get a little fancier. Using the canonical HRF will tell you how much the canonical HRF (convolved with your design matrix) needed to be scaled to account for your signal. But you might be interested in more detail - how much variance there was in the onset of the HRF, or how much in its length, or the true shape of the HRF for your subjects. Or you might not think the canonical HRF is a good enough fit to your subjects or your region of interest for you (and there's certainly evidence to make that thought reasonable).

In those cases, you may want to complicate your model a bit. In the extreme, you could not use any sort of a guess at an HRF, and instead directly estimate the shape of your HRF by separately estimating parameters for every timepoint following your stimulus - a finite impulse response (FIR), or deconvolution model. We'll talk about those in more detail later in the course. Alternatively, you might choose to model your neural activity as a linear combination of basis functions like sines and cosines - this will guarantee you can get an excellent fit to your true HRF and use a different HRF at every voxel, thus avoiding the problem of regionally-different HRFs. This is a Fourier basis set model. As an intermediate step between the FIR/basis set - type models and the pure canonical HRF models, you might try modeling your activation as a combination of two or three functions - say, the canonical HRF and its temporal derivative, or dispersion derivative. This will separately estimate the contribution of the canonical HRF and how much variability it has in time of onset (temporal deriv.) or shape (dispersion deriv.).

There are tradeoffs for using the more complicated models: for FIR and Fourier models, the interpretation of any given parameter value becomes very different, and it becomes much more difficult to design contrasts. Using the intermediate steps can get you back physical interpretability, but at the price of decreased degrees of freedom in your data without a guarantee of better fit overall - and, in fact, Della-Maggiore et. al find that the HRF w/ temporal derivative has significantly decreased power relative to the canonical alone in a typical experimental design. So use at your own risk...

3. When does it make sense to do a regionally-specific HRF scan?
*********************
If you're particularly interested in a region that's not primary visual cortex and you'd like to get a very good fit of your model to the data, it may make sense to try and get an HRF that's specific to a different region. The canonical HRF is derived from measurements in V1 of some subjects, and studies like Miezin et. al have demonstrated that between-region variability within a subject in one scan can be significant.

However, if you're worried about this, rather than taking an entirely separate scan or task to estimate impulse response in a region, it probably makes more sense to use an analysis type that models the response of each voxel separately - like an FIR or Fourier basis set model - which doesn't assume that you have the same HRF at every voxel.

4. When does it make sense to do a subject-specific HRF scan?
*********************
If you're looking to study intersubject variability, or if you're looking to improve the fit of your model by a good chunk and you can afford the extra time in the scanner. Several studies, like Aguirre et. al and Miezin et. al, have demonstrated there is a significant amount of variability between subjects in several parameters of the HRF - time to onset, time to peak, amplitude, etc.

Perhaps more importantly, the canonical HRF is based on measurements from normal, adult cortex. It's becoming clear that populations like children, the elderly, or patients of various kinds may have HRFs that differ significantly from the canonical. Particularly in any sort of between-group study in which these populations are being compared to normal subjects, it's crucial to ensure that any effect you see isn't driven by the difference in fit of the HRF between groups - you would expect the canonical HRF to provide a better fit (and hence more and larger activations) to normal adults than it would to patients or non-standard populations. In cases like these, using a subject-specific HRF may be necessary or at least desirable.

5. Which regions have particularly different HRFs?
*********************
Probably a lot of 'em. But in particular there are questions about the extent to which the canonical HRF or others measured from cortical neurons maps onto HRFs for subcortical structures like the basal ganglia. Definitive answers about this sort of thing await further study. Logothetis & Wandell (2004) discuss some reasons why regions might differ in HRF - from increased white matter density to differences in vascular density to, um, complicated physiological things. But they have one clear point: we know HRF can vary from region to region, and there is no accurate way currently to convert the absolute BOLD magnitude to any neural measure. Which means comparing absolute BOLD effect between regions, even nearby, is simply not justified theoretically. In their words:

*"It seems that with our current knowledge there is no secure way to determine a quantitative relationship between a hemodynamic response amplitude and its underlying neural activity in terms of either number of spikes per unit time per BOLD increase or amount of perisynaptic activity." - Logothetis & Wandell (2004)*

So be cautious about comparing absolute magnitude between regions...

6. Which populations have particularly different HRFs?
*********************
Surprising few studies have been published on this subject. Ongoing studies at Stanford (Moriah Thomason & Gary Glover) suggest children of at least a certain age have important differences in their HRF, and it's clear the same is true of elderly subjects. But at present, if you're interested in using any non-standard population of subjects, it's not a crazy idea to assume they have significant differences in their HRF from the standard canonical.

7. What's the difference between an 'epoch' HRF and an 'event' HRF?
*********************
If you're using AFNI/BrainVoyager/SPM2, there isn't any. So don't worry about it. But if you're using SPM99 or earlier, the story above about the canonical HRF is actually oversimplified. In SPM99 (and earlier), epoch-related and event-related studies had the same underlying design matrix form - the onset of a trial was marked with a 1, but the actual trial itself was all 0's. Event-related studies were simply modeled by convolving those with a canonical HRF, but epoch-related studies clearly needed to account for the length of the trial. So there was a separate, epoch-related, canonical HRF, that was also based off a mixture of gamma functions, but was specifically scaled to account for the length of the trial - so the HRF that was convolved with the design matrix was different for a 12-second epoch and a 30-second epoch. The epoch HRF generally looked like a wider, fatter canonical HRF, and represented a model of linearly summed HRFs over the course of the trial.

With a re-vamping in data structures, though, in SPM2, and further study, this difference was scrapped. Epochs are now modeled differently at the level of the origingal, pre-convolved design matrix, with 1's down the whole length of the trial, and events and epochs are convolved with the same canonical HRF. This seems provide an equally good or better fit to real data as well as simplifying many calculation aspects.

8. What relationship does the BOLD have to the underlying neuronal activity?
*********************
(This should probably be higher up the question list.) The BOLD signal is produced by an influx of oxygenated blood to a local area of neuronal activity, to compensate for increased energy usage. But neurons use energy for a lot of things, both pre- and post-synaptic: action potentials, increased membrane potentials, cleaning up neurotransmitter, putting out neurotransmitter, etc. In order to correctly interpret the BOLD in a given region, we need to know if it's caused by, say, increased firing (i.e., output) or increased post-synaptic activity (i.e., increased input) or some combination. What aspect of the underlying electrophysiology does the BOLD correlate with?

Logothetis & Wandell review a good deal of this work. Several electrophysiology studies have found tight correlations between BOLD and local field potential (LFP), a lower-frequency electrical measure summed over many neurons (but still with good spatial resolution). Similarly, fast ERP amplitude seems to vary linearly with BOLD (Arthurs & Boniface, 2003). Slow ERPS, which are believed to arise from _post_synaptic potentials, correlate with BOLD in parietal cortex (Schicke et al., 2006)). Some studies have also found linear relationships between spike rates and BOLD, and spiking activity likely also correlates with BOLD, although perhaps not as robustly (Logothetis et al., 2001).

All this goes to suggest that BOLD may originate less in actual neuronal spiking and more in low-frequency potentials or increased excitability of neurons: in other words, BOLD reflects input to an area more than output. Clearly, input and output are often correlated for neurons; excitatory input will increase spiking outputs. But they aren't always; if this hypothesized connection is true, it means an increased BOLD could reflect increased inhibitory input to a region, or summing of both inhibitory and excitatory inputs (resulting in no change in spiking). Logothetis & Wandell also cite examples where BOLD might be different from single-unit recording - e.g., an area may appear highly direction-sensitive with BOLD, even if single-unit recordings show it not to be, because it is highly interconnected with a very direction-sensitive area. Attention effects, which are difficult to find in V1 with single-unit recordings but show up in fMRI, might be another example. So we should have caution in using models that require BOLD signal to directly index increased spiking outputs.

This is not to say it's impossible to map single-unit firing onto BOLD signal; retinotopy in visual cortex, for example, happens at the single-unit level, and is easily detectable with BOLD. But merely note: many factors, from input activity to vascular density (see above), etc., can affect regional BOLD response.

9. How does the hemodynamic response change with stimulus length?
*********************
In very nonlinear ways. See Glover (1999) (and Logothetis & Wandell, 2004). You're on very shaky ground if you attempt to model stimuli longer than 6 sec by convolving a standard HRF with a boxcar. You may be better off using stimuli separated by longer times or shorter stimuli.


Jitter
===================

Jittering is heavily mixed in with experimental design and setting your scanning parameters, so be sure to check out the other design-related pages: `Design`_, `Scanning`_ and `Physiology and fMRI`_

1. What is jittering?
*********************
It's the practice of varying the timing of your TR relative to your stimulus presentation. It's also often connected to, or even identified as, the practice of varying your inter-trial interval. The idea in both of these practices is the same. If your TR is 2 seconds, and your stimulus is always presented exactly at the beginning of a TR and always 10 seconds long, then you'll sample the same point in your subject's BOLD response many times - but you might miss points in between those sampling points. Those in-between points might be the peak of your HRF, or an inflection point, or simply another point that will help you characterize the shape of your HRF. If you made your TR 2.5 seconds, you'd automatically get to sample several other points in your response, at the expense of sampling each of them fewer times. That's "jittering" your TR. Alternatively, you might keep your TR at two seconds, and make the time between your 10-second trials (your inter-stimulus interval, or ISI) vary at random between 0 and 4 seconds. You'd accomplish the same effect - sampling many more points of your HRF than you would with a fixed ISI. You'd also get an added benefit - you'd "uncover" a whole chunk of your HRF (the chunk between 10 seconds and 14 seconds) that you wouldn't sample at all with a fixed ISI. That lower portion can help you better determine the shape of your whole HRF and find a good baseline from which to evaluate your peaks. This added benefit is why most people go the second route in trying to "jitter" their experiment - varying your ISI gets you all the benefits of an offset TR, plus more.

2. Why would I want to do this in my experiment?
*********************
If all you care about is the amplitude of your response, you probably wouldn't. In this case, you're assuming a certain shape to the hemodynamic response, and all you care about is how "high" the peak of the HRF was at each voxel for each condition. You'd want a design with very high statistical power - the ability to detect amplitude. On the other hand, you might not want to assume that every voxel had the same HRF shape for every condition. If you'd like to know more about the shape, without assuming anything (or less than everything, at least), you need a design with high statistical efficiency - the ability to accurately estimate shape parameters, without assuming a shape.

Varying your ISI is a strategy to increase the efficiency of your estimates at the expense of your power. Clearly, because you'll be sampling each point of your HRF fewer times, you'll necessarily have less confidence in the accuracy of any given estimate. But because you'll have so many more points to sample, you'll have much more confidence about the true shape of your HRF for that condition. This is critical when you believe your experiment may induce HRFs of different shape in different region, or if knowing the shape (lag, onset time, offset time, etc.) of your response is important (as it is in mental chronometry). With a variable-ISI design, you can run a rapid event-related design and pack many more trials into a given experimental time than you would for a fixed-ISI design that sampled the whole HRF, or you can sample much more of the HRF than a fixed-ISI design could with the same number of trials.

3. What are the pros and cons of jittering / variable-ISI experiments? When is it a good/bad idea?
*********************
Variable-ISI experiments are a way of making the tradeoff between power and efficiency in an experiment. Fixed-ISI designs are extremely limited in their potential efficiency. They can have high power by clustering the stimuli together: this is a standard block design. But in order to get decent efficiency in an experiment, you need to sample many points of the HRF, and that means variable ISI. Any experiment that needs high efficiency - say, a mental chronometry experiment, or one where you're explicitly looking for differences in HRF shape between regions - necessarily should be using a variable-ISI design. By contrast, if you're using a brand new paradigm and aren't even sure if you can get any activation at all with it, you're probably better off using a block design and a fixed ISI to maximize your detection power.

From a psychological standpoint as well, the big advantage of variable-ISI designs is that they seem far more "random" to subjects. With a fixed ISI, anticipation effects can become quite substantial in subjects just before a stimulus appears, as they catch on to the timing of the experiment. Variable ISIs can decrease this anticipation effect to a greater or lesser degree, depending on how variable they are. 

4. How do I decide how much to jitter, or what my mean ISI should be?
*********************
Great question. Depends a lot on what your experimental paradigm is - how long your trials are, what psychological factors you'd like to control - as well as what type of effect you're looking for. Dale et al. lays out some fairly intelligible math for calculating the potential efficiency of your experiment. Probably even easier, though, is to use something like Tom Liu's experimental design toolbox. Once you're within the ballpark for the type of paradigm you like, these tools can be an invaluable way to optimize your design's jitter / ISI variation, and are highly recommended for use.

5. But how do I get better temporal resolution than my TR?
*********************
Simple: don't always sample the sample points of your response. If you always sample the BOLD response 2 seconds and 4 seconds and 6 seconds after your stimuli are presented, for your whole experiment, you'll have a very impoverished picture of the shape of your HRF. But if, for example, you sampled 2 sec. and 4 sec. and 6 sec. post-stimulus for half the experiment, then cut one second between trials and sampled 1 sec. and 3 sec. and 5 sec. for the rest of your experiment - why, then, you'd have a better picture. The cost, of course, is reduced power and expanded confidence intervals at the points you've sampled.

With a good picture of the shape of your HRF, though, you could then compare HRFs from two different regions and see which one had started first, or which one had reached its peak first. If HRF timing is connected in some reliable way to neuronal activation, you then don't need to sample the whole experiment at a super-fast rate - you could infer from only a limited-sample picture of one part of the HRF where neuronal activity had started first and where it had started second, which allows you to rule out certain flows of information.


MATLAB Basics
===================

Here's a very quick intro to some of the basic concepts and syntax of MATLAB. The hope is that after covering some of these basics, you'll be able to troubleshoot SPM and MATLAB errors to some degree. Experienced MATLAB programmers may find most of this redundant, but there's always more you can pick up in MATLAB, so please, if you experience folk've got tips and tricks to suggest, add them onto these page!

This part looks specifically at the most basic of basics. For info on how to read m-files and m-file programming concepts, check out `MATLAB Programming`_. As well, `MATLAB Paths`_ answers questions about modifying your search paths, and `MATLAB Debugging`_ touches specifically on strategies for troubleshooting - how to get more information, what parts of error messages to look for, and how you can dig through a script to find what's gone wrong.

If you take nothing else away from this page, take this: The MATLAB tutorial, contained within the program's help section, is terrific. It'll teach you a lot of the basics of MATLAB and a lot of things you wouldn't think to learn. The progam help is always a great place to go for more info or to learn something new. Nothing will help your troubleshooting and programming skills like curiousity - when you hit something you don't know or recognize, spend a couple minutes trying to figure out what it is, with the MATLAB and SPM help. If your help within the program isn't working, it's all online at the `Mathworks website <http://www.mathworks.com/access/helpdesk/help/techdoc/MATLAB.shtml>`_.
As well, if you type help command into MATLAB, where command is some MATLAB function name, you'll almost always get a quick blurb telling you something about the function and how it's supposed to operate. It's invaluable for reference or learning on the fly.

MATLAB Language
*********************
Here are some quick basics on MATLAB code and what it all means. You don't need to be expert on all of this, but it may help you understand what an error message or piece of script means - a sort of Rosetta stone, if you will.

**Variables**

The very very first thing about MATLAB to understand is what a variable is. A variable in MATLAB is like a variable in algebra - it's a name given to some number. In MATLAB, you can name any single number (like a = 4) or multiple numbers at once, in which case the variable is a vector or an array of numbers (like a = [1 2 3 4]). A single number is called a scalar variable in MATLAB, but a single variable name can represent a huge array of numbers of arbitrary size and dimension. Creating a variable or manipulating it is done with = statements, where the left side of the = is your variable name, and the right side is the value you want it to take. If the name doesn't already exist, an = statement will create it - something like my_variable = 6005. If the name is already taken for a variable, an = statement can be used to re-assign that variable name to something else entirely, or simply manipulate the values within that variable. You can change a whole variable at once or simply a few of the numbers (or characters, or whatever) within it.

**Workspace**

When you start MATLAB, it opens a chunk of memory on the computer called the "workspace." When you create a variable, it exists in the workspace, and it stays there until you clear it, or until you re-assign that variable name to another value (which you can do freely). Only variables in the workspace can be used or manipulated. You can easily tell what variables are present in the workspace with the command whos . You can save a whole workspace to the disk with the command save filename; this will create a binary file with a .mat extension in your present working directory (more on the working directory later). That .mat file contains all the variables you've just saved, and you can clear the workspace and then load those variables again with the command load filename. You can clear the whole workspace at once with clear or clear any individual variable name with clear variable.

**Array notation**

Sets of numbers (or characters, or other things) in MATLAB are arranged in arrays - basically rectangles of rows and columns of numbers. Arrays can be only a single number - those are called scalars. They can also be single rows or columns - those are called vectors. Two-dimensional arrays (with multiple rows and columns) are pretty common, but arrays can also be three-dimensional, or even four- or more-dimensional. If you want to access a single element of the array, you use array notation, which puts parentheses after the variable name to specify which element. So if I have a 4-by-4 array called a, a(3,2) refers to the element in the third row, second column. Array notation always refers to row first, column second, other dimensions afterwards in order. You can type a(3,2) by itself in MATLAB and it'll tell you what's in that spot, or you can use = to reassign that number, with a command like a(3,2) = 16. If you have a vector, you can use only one number in the parentheses - something like a(4) = 56. You can make arrays by using brackets. a = [1 2 3 4] will make a four-element row vector - a 1x4 matrix. Separating the elements by spaces will put them in different columns on the same row. Separating them by semicolons will put them in different rows in the same column, so a = [1;2;3;4] will make a column vector, or a 4x1 matrix. You can combine the two methods. a = [1 2 3 4; 1 2 3 4; 1 2 3 4; 1 2 3 4] will make a 4x4 matrix, with 1 2 3 4 on each row. Using the colon vector to specify a range of numbers can be a quick way to build matrices this way. There are also several commands, like eye, ones, zeros, and others, that build specify types of matrices of a specific size automatically.

**Variable types**

There are several types of variables you can use in MATLAB. These include various kinds of numbers - integer, floating point, complex - as well as alphanumeric charactetrs (arrays of characters can be treated as strings). A given standard array can only hold one 'type' of thing - an integer array can only have integer values, a character array can only have characters as values, etc. Sometimes, though, you'd like to have more than one type of information in a given array, lumping several different-sized number arrays together, or lumping some numbers and strings and so forth together into one unit. For those, you can use cell arrays and structures, two specialized forms of variables.

**Cell arrays**

Cells array are like normal arrays, but each element of a cell array can contain anything - any number type, any other array, even more cells. Cell arrays are accessed like normal arrays, but using curly braces instead of standard parentheses, so if I have a cell array c, I might say c{3,2} = 16, followed by c{3,3} = 'cat'. I can refer to the cell itself with standard parentheses - so c(3,3) is a cell - or the contents with the curly braces - so c{3,3} is a character array above. I can refer to the cell's contents in an expression like this by following the curly braces with the standard array notation. If I have the above array, c{3,3}(2) would be a single character - 'a'. You can make cell arrays simply by using the curly braces: mycell{1,1} = 'hello' would make a single cell containing the string 'hello'. The command cell will also make empty cell arrays. There are also various utilities to convert arrays back and forth between cell arrays and standard arrays.

**Structures**

Structures are like regular variables that are made up of other variables. So a single structure contains several 'fields,' each of which is a variable in its own right, with a name and a particular type. Fields can be of any type - numbers, letters, cells, or even other structures. SPM makes enormous use of structures in its data organization, particularly structures nested within other structures. If you have a structure called s, it might have subfields called data and name, and you could access those with a dot, like so: s.name = 'Structure, or s.data = [1 2 3 4]. Nested structures are accessed the same way; you could access a sub-subfield by something like s.substructure.smallstruct.data = 3. You can make a structure with the struct command, where you specify fields and values, or you can create them simply by naming a new variable and using a dot - so if there was no variable called s, the command s.name = 'Structure' would create a structure called s with a single field called name. You can add fields just by naming them in the same way, and remove them with the rmfield command. Structures can be put in arrays or vectors, so long as all the structures in the array or vector have the exact same field names and types. So you could reference s(4).data if you had 4 structures all in a vector called s. Typing the name of a structure will always tell you all its fields and what they are.

**The colon operator**

Last among the real basic things in MATLAB is the colon operator, which operates as shorthand for a whole range of numbers at once. The expression 1:10 is shorthand for "every number between one and ten, inclusive, counting by ones". You can specify a different number to count by simply by putting a different number between two colons, so the expression 1:2:10 is every number between 1 and 10, counting by 2s (1, 3, 5, 7, 9). You can use the colon operator to make values for arrays: the command a = 1:10 makes the row vector [1 2 3 4 5 6 7 8 9 10]. You can also use it in indexing other arrays, where it's particularly powerful. If a is a 4x4 array [1 2 3 4; 1 2 3 4; 1 2 3 4; 1 2 3 4], the expression a(2:4, 2:4) gives me back a new matrix which is the second through fourth rows and columns: [2 3 4; 2 3 4; 2 3 4]. I can use the colon operator by itself as shorthand for "all rows" or "all columns;" with the 4x4 a array, the expression =a(:,1:2) would give me [1 2; 1 2; 1 2; 1 2].

**Using all notation together**

It'll take a while if you haven't programmed before, but eventually you'll learn to use all of these notations together, which make the language particularly powerful. An expression like Sess{3}.U(4).C.C(1:10, :) may look horrible on the face of it, but all it's doing is picking out a small submatrix; that submatrix is embedded in (reading right-to-left): a larger matrix (Sess{3}.U(4).C.C) inside a structure (Sess{3}.U(4).C) inside a vector of structures (Sess{3}.U) inside a cell (Sess{3}) which is part of a cell array (Sess). That submatrix would be the 1st through 10th rows of C and all of its columns. When in doubt on what an expression means, use whos and type the names of variables and fields to try and figure out what they are. Read backwards (from right to left) and try and work out the organization of the expression that way.

**Miscellaneous stuff**

The variable name ans is built into MATLAB as shorthand for "the result of the last thing you typed in." You can assign the results of commands to other variables, like b = a(1:3, 1:3), but if you don't and just type a(1:3, 1:3), the result of that command (a 3x3 matrix) will be given the name ans. The thing to be aware of is that the next time you type a command with unassigned output - say the next thing you typed was a(2:4,2:4) - the variable name ans would be reassigned to take on the value of the latest result. Whatever was in ans before is lost! So be careful assigning your output to variable names. And if you get something in ans that you want to keep, put it in another variable right away, by saying something like myvar = ans.

Generally, MATLAB will report the result of your command to the screen, telling you what the new value of your result variables was. If you don't want it to do that - because, for example, your result will be a 2000x2000 matrix - you can put a semicolon at the end of the line and MATLAB will suppress its output. So a = ones(2000,2000); will generate a 2000x2000 matrix of all ones, but it won't print that matrix to the screen. In m-files, you'll see almost every line ends with a semicolon so that you're not barraged during a function or script's execution by results whipping by.


MATLAB Debugging
*********************

This page looks specifically at how to troubleshoot MATLAB problems - how to get more information, how to read and interpret error messages, and how to dig through m-files to find what's gone wrong. For the most basic of basics, check out `MATLAB Basics`_. As well, `MATLAB Paths`_ answers questions about modifying your search paths, and `MATLAB Programming`_ talks about how to read m-files and how to program with them.

Troubleshooting SPM and MATLAB
----------------------

So there you are, tootling along estimating your model or coregistering your anatomy or what have you. And then, boom: computer beeps, error message pops up, everything stops, fire and brimstone come from the sky. What can you do about it? Lots, as it turns out. Armed with a sense of the MATLAB basics and understanding of MATLAB code, you can often divine a lot about what's going on in your error from your crash - often enough to fix the problem, but at least enough to narrow down the possibilities. The two crucial entry strategies to troubleshooting are: knowing where the problem happened, which frequently boils down to interpreting your error messages, and knowing what was happening when the problem hit, which relies on MATLAB's debugging package. We'll tackle them separately.

**Non-errors**

Before we jump into errors, a quick word about non-error problems. Warnings sometimes crop up in MATLAB, and they allow the program to continue. They're not generally anything worth worrying about - they'll let the program run - but they're worth noting, and trying to understand what they mean, particularly when they precede an error. The programmer puts warnings in for a reason, to notify you of something weird happening, so listen to her...

Also, people sometimes ask whether they can tell if an SPM program is crashed or just running for a long time. In general, MATLAB rarely crashes outright without generating some kind of error message, so if something's stopped responding and you've got a "busy" message but no error message, it's probably still working. If it seems like it's been going for an inordinate amount of time, though, check in with somebody.

**Where did the error happen? or, Everything you need to know about error messages**

SPM errors can happen in terribly uninformative fashion. Everyone who's used SPM has seen the infamous "Error during uicontrol callback," which is a generic MATLAB error that means "Something went wrong in the function called by pushing whatever button you just pushed." Fortunately, MATLAB has relatively well-designed error-detection and bug-tracking facilities. You can learn a great deal about your error just by interpreting your error message.

Sometimes in SPM you'll get a full error message of several lines, but oftentimes you'll get the one-line "uicontrol callback" error. However, if you close the panel that's home to the button that generated the error, you can often get the full underlying error message. So if you hit "volume" in the results section and get a uicontrol callback error, closing the results panel will usually generate further information about the error. You may also be able to generate more information simply by going to the MATLAB window and hitting return a couple of times. Always try and get a full error message when you have an error! Even if you can't figure out the problem, it's almost totally useless to another troubleshooter if you come to them and say, "I have an error!" or "I have a uicontrol callback error!" without any more information about the problem. Gather as much information about the error as you can - the MATLAB window is your friend.

MATLAB error messages are of varying helpfulness, but they all follow the same structure, so let's take a look at one:

.. code-block:: none

    ??? Error using ==> spm_input_ui Input window cleared whilst waiting for response: Bailing out!
    Error in ==> /usr/local/MATLAB/R2014a/toolbox/spm12/spm_input.m
        On line 77 ==> [varargout{:}] = spm_input_ui(varargin{1:ib-1});
    Error in ==> /usr/local/MATLAB/R2014a/toolbox/spm12/spm_get_ons.m
        On line 146 ==> Cname{i} = spm_input(str,3,'s',sprintf('trial %d',i),...
    Error in ==> /usr/local/MATLAB/R2014a/toolbox/spm12/spm_fMRI_design.m
        On line 225 ==> [SF,Cname,Pv,Pname,DSstr] = ...
    Error in ==> /usr/local/MATLAB/R2014a/toolbox/spm12/spm_fmri_spm_ui.m
        On line 289 ==> [xX,Sess] = spm_fMRI_design(nscan,RT);
    ??? Error while evaluating uicontrol Callback.

SPM errors will often look like this one: a "stack" of errors, although the stack may be bigger or smaller depending on the error. So what does it mean?

First, the order to read it in. The top of the stack - the first error listed - is the most immediate error. It's what caused the crash. MATLAB will tell you what function the error was in - in this case, it's spm_input_ui - and some information about what the error was. That error message may be written by the function's programmer (in which case it's often quite specific) or be built-in to MATLAB (in which case it's usually not). Sometimes that top error may also give you a line number on which the error was generated within the function.

The rest of the stack is there to tell you where the function that crashed was called from. In this case, the next one down the stack was in spm_input - so our error was in spm_input_ui, which was called by spm_input. The line in spm_input that calls spm_input_ui is highlighted for us - it's line 77 within spm_input.m. The next error down tells us that spm_input was itself called from within spm_get_ons, on line 146, and so on. The last error tells us that the very first function that was called was spm_fmri_spm_ui - that's the function that was called when we first hit a button.

From only this limited information, you can often tell a lot about the error. One important distinction to draw is whether the function where the error happened was an SPM function or a MATLAB function. Built-in MATLAB functions are generally pretty stable, and so if they crash, it's usually because you, the user, made a mistake somewhere along the line. 

When the function that generated the error is an spm function, you can usually tell - it'll have the prefix 'spm_' or generally sound like something from fMRI analysis and not MATLAB generally. In this case, the error is sometimes in the code and sometimes a mistake earlier in your analysis. The error messages can sometimes tell you enough to figure out what's going on: perhaps it says that there's an index out of range, and you realize it's because you're trying to omit the 200th scan from a session that only has 199. Just looking at the line that generated the error and what the actual error message says can give you a lot of info off the bat. Study the message carefully and look at the function stack it came from and see if you can figure out what the program was doing when it crashed.

It's not always enough to look at the problem after it's crashed, though. Sometimes you need to figure out what's happening when it actually crashed, and in that case, you need MATLAB's debugging package.

**MATLAB Debugging**

MATLAB's debugging package is pretty good. Hard-core debugging is a skill that's acquired only after a lot of programming, but even someone brand-new to reading code can use some debugging techniques to make sense of what's going on with their data, and hopefully learn some details about MATLAB code at the same time.

You'll need to be running the graphical version of MATLAB to take real advantage of MATLAB's debugging; without it the text editor won't open and you won't be able to see the details of the function you're debugging.

The starting point for any bug-finding expedition is to answer the question, "What was happening in the program when it crashed?" The hope is answering that will tell you why it crashed, and how to keep it from crashing again. When errors ordinarily occur in MATLAB, they end the execution of the program and close the workspace of the error-ing function, losing all its variable and so forth. The debugging package allows you, instead, to wait for an error and then "freeze" the program when one happens, holding it still so you can examine the state of all its variables and data.

If you want to try debugging an error, figure out when your error happens first (during model estimation or when you push "load" or whatever). Then type "dbstop if error" at the MATLAB prompt. This will engage the debugger in "waiting" mode. Then run your program again and do whatever caused the error. This time, you'll get an error message, but the prompt will change to K>>, and a text window will pop open containing the function that caused the error. The K prompt means you have Keyboard control - the program is frozen, and you can poke around as much as you like in it. To end the program when you're done debugging, type "dbquit" at the K>> prompt, and the program will end.

Once you have the program frozen, it's time to look around. Check out the line that generated the error and see if you can figure out what the error message means. If it's called by an error function, is it inside an if block? If so, what condition is the if block looking for? If the error is caused by an index out of range, can you figure out what the index variable is? What is its value? What is the array that it's indexing? What sort of information is in there? Type whos to figure out what variables are currently in the workspace and what types they are, and type the name of variables to find out their values.

It may not be immediately clear why a variable has the value it does or what it represents. Try reading line-by-line backwards through the function and find out when the variable got its current value assigned. Where did it come from? Are there any comments around it telling you what it is? Are there any comments at the top of the function describing it?

Sometimes a variable may have a strange value passed into it by whatever function called the error-ing function. The MATLAB debugger has frozen that function as well, and you can move 'up' into its workspace to take a look around in the same way. Type "dbup" at the K>> prompt, and you'll switch into the workspace of the function that called the current one. You can go up all the way to the base workspace, and "dbdown" will let you switch back down through the stack. Remember that each workspace is separate - their variables don't interact, at least when all the m-files are functions.

There are other debugging commands as well, more useful for you in writing your own scripts. Check out the MATLAB help for keyboard, dbcont, dbstep and things like that.

There's not more specific debugging advice to give, besides go explore the function. Every error is a little different, and it's impossible to tell what caused it until you get into the guts of the program. Good luck!


MATLAB Paths
*********************

This Part looks specifically at how to read and modify your search path. For the most basic of basics, check out `MATLAB Basics`_. As well, `MATLAB Programming`_ talks about how to read m-files and program with them, and `MATLAB Debugging`_ touches specifically on strategies for troubleshooting - how to get more information, what parts of error messages to look for, and how you can dig through a script to find what's gone wrong.

Paths
----------------------
Every time you type a name in MATLAB - of a variable, a function, a script, anything - the program goes through a certain procedure to figure out what you're referring to. It'll first look in its workspace for variables of that name; if it doesn't find anything, it will assume you're trying to refer to a function, script, or .mat file. MATLAB always looks first in the present working directory for the .m file or .mat file of the given name, but if it doesn't find one there, it has a specified sequence of directories it looks in.

This directory list is called the path. It's similar to the Linux concept of the same name. Maintaining and manipulating your path can be crucial to running MATLAB programs and keeping track of different versions of functions. Fortunately, MATLAB makes it pretty easy to work with paths.

You can always look at your path in MATLAB by typing path; MATLAB searches the output list from top to bottom. The easiest way to work with your path is by typing "pathtool" at the MATLAB prompt. That'll open up a graphical interface that will display your current path, let you easily add or remove directories, or re-order what's in there already.

If you're curious where on your path a particular m-file is, you can type which filename; this will tell you the location of the first copy of that file MATLAB hits on the path, and hence which version of the file it will run. If you type open filename and filename is on your path, open will open the first copy it hits on the path. These two commands can be super helpful in figuring out whether you've got the right version of code, as well as figuring out where functions are located.

In general, any manipulations of the path you do are good only for your current session of MATLAB; when you shut it down and restart it, you'll go back to your default path. The default path in MATLAB is stored in a file called pathdef.m. But you can change your own default path! All you need is a startup.m file.

Startup.m files
----------------------

When you launch MATLAB, it will automatically search for an m-file called startup.m in a subdirectory of your home directory. If it doesn't find a startup.m file there, no biggie - nothing happens. If a file called startup.m exists there, though, it will automatically run that file. This can be a super useful way for you to customize MATLAB for your own use. There are a number of commands you can run in MATLAB to change output or parameters, but one obvious thing to do is customize your path.

The command addpath('directoryname'); in a startup.m file will automatically ensure that directoryname is the first directory on your path whenever you start MATLAB. If you have several addpath statements in there, they'll run in order, adding to the top each time, so the last addpath statement specifies the first directory on your path. This is really helpful if you want to keep your own MATLAB code (or MATLAB data) available above anything else. Simply add the directory where you keep the code or data to the top of your path, and you'll always be able to run it, no matter where your present working directory is.


MATLAB Programming
*********************

This part looks specifically at how to read m-files and m-file programming - functions, scripts, etc.. For the most basic of basics, check out `MATLAB Basics`_. As well, `MATLAB Paths`_ answers questions about modifying your search paths, and `MATLAB Debugging`_ touches specifically on strategies for troubleshooting - how to get more information, what parts of error messages to look for, and how you can dig through a script to find what's gone wrong.

Functions
----------------------
Once you've figured out some basic ways to create and refer to your variables and store some information in them, you probably want to manipulate that information. That's where functions and scripts come in. 

A function is essentially a shorthand name for a whole set of commands. They're stored in m-files - text files with .m extensions. Any file with a .m extension you can generally read with a standard text editor, and it'll just contain a whole set of MATLAB commands. Conventionally, you put only one command per line of the file, and when you run the function, MATLAB just reads through the file top to bottom, executing each command in order. You run a function simply by typing the name of its m-file, assuming that m-file is in the MATLAB path.

Functions often will take some data as input and spit back out some data as output. Functions are referred to kind of like arrays, but instead of supplying indices, you supply input variables. So saying spm_vol(VY) is calling the function spm_vol and passing it the input variable VY. If you said mydata = spm_vol(VY), you'd be saying you wanted whatever output that spm_vol chose to spit out to be stuck into the variable mydata.

Scripts
----------------------
A .m file can contain either a function or a script. The difference between the two is subtle but important; it has to do with what workspace you have access to when you're executing the commands inside your m-file. You can easily tell the two apart; functions always have the function command as their first executable line, which will specify the name of the function and what its input and output are. Scripts don't have the function command - they just launch right in.

The difference between the two is in the workspace. When you run a function, it opens its own, clean workspace. Any variables you have in the workspace when you run the function can't be accessed inside the function, unless you pass them in as input variables. When it ends, it closes its workspace and any variables there that aren't designated as output variables are lost.

Scripts, by contrast, operate in the same workspace from which they're called. So any variable you have in the workspace when you call a script can be manipulated by the script, and any variable created in the script will be left in the workspace when the script ends.

Reading .m files
----------------------
If you're running the graphical version of MATLAB, you can open the built-in text editor with the command open filename. The single best thing you can do to learn and teach yourself about the inner workings of SPM (or other MATLAB packages) is to open up particular functions and read through them to see what they're actually doing. Reading them can seem like reading Greek a lot of the time, but it's a skill you can learn, and it'll help you understand what's happening to your data better than anything else. Unless you can read MATLAB code, you'll always be a little in the dark about what's being done in your analysis. You don't need to know how to program the code - just to understand more or less what it's doing.

Reading code is relatively straightforward in some ways - you simply start from the top and go through line-by-line to see what each successive line does. But there are some things to keep in mind on the way:

* Lines that start with % are "comments." Anything after a % on a line is ignored by MATLAB (even if there was real code before the % on the line), and so comments are used by programmers to explain what's happening in a file. Most functions have some overview comments at the top of the file, to try and explain what's happening overall and hopefully some of the data structures and variables in use. Good programmers also comment throughout the function, to point out different sections of code and help explain what's happening in each chunk, or why they chose to use a particular strategy. Read the comments! They can be super helpful in understanding what's going on.
* If you see a function name and don't know what it does, check the MATLAB help! There's a lot of information in there about what every function under the sun does. Many MATLAB functions are obviously named, but not all. If it's not a MATLAB function, try typing help function-name at the prompt anyways - a lot of times something will come up.
* Certain statements control the flow of execution through the program - if and for are the biggies, but there are others like switch or while, too. They're generally paired with an end statement; anything in between the if or for and the end is under the control of that flow statement. Generally those blocks of code are indented, for readability. They're often nested within each other, too. An if statement specifies some condition to test; if the condition is true, then the block of code before the paired end will be executed. If the condition is false, that code won't run at all. A for statement is used to execute a block of code several times. For statements specify some index variable - often named i or j, but sometimes other things - and specify the range of values it will take on. It'll then loop over the code before the paired end statement and execute it for each value of the index variable. So for i = 1:10 will make a loop that will run 10 times, with i becoming one number bigger each time. This is often used to walk through a whole vector or array of numbers and do something to each element, using the index variable as the index into the array.
* If you get stuck, ask somebody! There are experienced programmers out there who can help.


Mental Chronometry
===================

Also check out `Jitter`_ for more info on variable-ISI experiments...

1. What is mental chronometry?
*********************
As important (or moreso) than finding out where your activation happened is finding out when it happened. Where did the information flow during the processing of your stimuli? Which structures were active before other structures? Which structures fed output into other structures, and which structures processed end results? Such questions suggest a fairly crude schematic of brain processing, but still a useful one: if you could answer all of those accurately about a given task, tracking the millisecond-by-millisecond flow of information through the brain, you'd have a much fuller picture of the information processing than from a static activation picture. So mental chronometry experiments attempt to attack those questions. First done with reaction time data, using experiments that would add or remove certain stages of a task and find out whether reaction time was sped or stayed the same or what, chronometric experiments have moved into fMRI, with moderate success. Formisano & Goebel (below) review recent developments in fMRI chronometry, with a thorough overview of potential pitfalls. They examine several studies in which flow of information is pinned down with a couple hundred milliseconds - not quite the temporal resolution of EEG, but much better than previously thought could be achieved with fMRI.

2. But how do I get better temporal resolution than my TR?
*********************
Simple: don't always sample the sample points of your response. If you always sample the BOLD response 2 seconds and 4 seconds and 6 seconds after your stimuli are presented, for your whole experiment, you'll have a very impoverished picture of the shape of your HRF. But if, for example, you sampled 2 sec. and 4 sec. and 6 sec. post-stimulus for half the experiment, then cut one second between trials and sampled 1 sec. and 3 sec. and 5 sec. for the rest of your experiment - why, then, you'd have a better picture. The cost, of course, is reduced power and expanded confidence intervals at the points you've sampled.

With a good picture of the shape of your HRF, though, you could then compare HRFs from two different regions and see which one had started first, or which one had reached its peak first. If HRF timing is connected in some reliable way to neuronal activation, you then don't need to sample the whole experiment at a super-fast rate - you could infer from only a limited-sample picture of one part of the HRF where neuronal activity had started first and where it had started second, which allows you to rule out certain flows of information.

3. How does variable ISI relate to mental chronometry?
*********************
In order to do a mental chronometry experiment, you need to absolutely maximize your statistical efficiency - your ability to pin down the shape of your HRFs. You're going to be comparing HRFs from several regions, and you need to look for the differences between them, which means you can't assume much (if anything) about what shape they're going to be, or else you're going to bias your results. Assuming nothing about your HRF and still getting a good idea of its shape, as Liu describes, means you need an experiment with very high efficiency - and, as Dale demonstrates, those are precisely those designs with variable ISIs. Only by randomizing (or pseudo-randomizing) your ISIs can you pack enough trials into an experiment for sufficient power and still have enough statistical flexibility to get a good look at the shape of your HRFs.

4. What are some pitfalls in mental chronometry with fMRI, then?
*********************
The big one is a crucial assumption mentioned above: that HRF timing is connected in some reliable way to neuronal activation. We assume you're not interested so much in the HRF for its own sake, but rather as an indicator of neuronal activity. So let's say you get two HRFs, one from visual cortex and one from motor cortex, and the motor HRF starts half a second later than the visual. Is that because neuronal activity started half a second later in motor cortex? Or is it because the coupling between neuronal activity and BOLD response is just slower in motor cortex in general? Or is it because the coupling in that particular subject just happens to be looser for motor than for visual - and maybe it'll be different for your other subjects! Clearly, no matter how good a look you get at your HRF, questions like these will dog your chronometric experiment unless you're careful about validating your assumptions. Several excellent studies have examined the issue of variability of HRF between regions, subjects, and times, and those studies are crucial to check out before drawing conclusions from this sort of data.

Of course, other more mundane issues may well torpedo chronometric conclusions: if you don't have high enough efficiency in your experiment, you won't be able to distinguish one HRF's shape from another with high accuracy, and you'll have a hard time telling which one started first or second anyways.

5. If I want to do a mental chronometry experiment, how should I design it?
*********************
Chronometric experiments depend crucially on determining HRF shape. So start with maximizing that - you need a design that will absolutely maximize your efficiency, no matter what the power cost. M-sequence or permuted block designs are good ways to start (see Liu). It should be obvious that with designs like that, you need experimental tasks that will generate fairly reliable activations; your experiment will suffer in terms of power from its focus on finding HRF shape and not using shape assumptions. Choosing a task with a reasonably long latency is also important - even with the best possible design, fMRI noise is such that resolution below a couple hundred milliseconds is simply not possible for now. So if your task only lasts half a seconds, you may not be able to get much information about the chronometric aspects with fMRI. As well, in an experiment like this, having more samples is always better - so you want to have the shortest possible TR. If you can focus your experiment to a smaller segment of the brain than the whole thing, you can get a good number of slices and still have very fast Trs.

One thing to try and avoid in doing chronometry is to toss it in as a fishing-expedition analysis: if your experiment isn't designed with doing chronometric analysis in mind, you'll almost certainly have trouble finding reliable latency differences in your subjects. Unless you've got an eye on this from the start, it's probably not worth doing. But if you do, you can get some pretty sweet looks at the temporal flow of activation and information around your subjects' heads.


Normalization
===================

1. What is normalization?
*********************
Inconveniently, brains come in all different sizes and shapes. But the standard statistical algorithms all assume that a given voxel - say 10, 10, 10 - samples exactly the same anatomical spot in every subject. So you'd really like to be able to squash every subject's brain into exactly the same shape and exactly the same space in your image. That's normalization - it's the process by which pictures of subjects' brains are squashed, stretched and squeezed to look like they're about the same shape

2. How does normalization work?
*********************
A lot like realignment. Normalization algorithms, just like realignment algorithms, search for transformations of the source image that will minimize the difference between it and the target image - transformations that, as much as possible, will make the source image 'look like' the target template. The difference is that realignment algorithms restrict themselves to rigid-body transformations - moving and turning the brain, but not changing its shape. Normalization algorithms allow nonlinear transformations as well - these actually change the shape of the brain, squeezing and stretching certain parts and not other parts, to make the source brain 'fit' the target brain. Different types of nonlinear transformations can be applied - some use sine/cosine basis functions, some use viscous fluid models or meshes - but all normalization can be thought of this way.

An important point about normalization is that any algorithm, if allowed to make changes on a fine enough scale, can precisely transform one brain into another, exactly. Sometimes, though, that's not what you want - if you're interested in looking at differences of gray matter in children vs. in adults, you'd like to normalize the general anatomy, but not at such a fine scale you remove exactly the difference you're looking for! Other times, though, you'd love to match up every point in a subject's brain exactly with the identical point in another subject's brain. Care should still be taken, though - normalization algorithms can align structural anatomy precisely, but can't guarantee the subjects' functional anatomies will align perfectly.

3. Why would I want to normalize? What are the drawbacks and/or advantages?
*********************
The advantage is simple: Brains aren't all the same size and shape. The simplest and most widespread methods of statistical analysis of brain data is to look each voxel across all your subjects and compare some measure in that voxel. For that method to be reasonable, equivalent voxels in each subject's images should match up with equivalent locations in each subject's brain. Since brain structures can be quite variable in size and shape, we need some way to 'line up' those structures. If you want to do any kind of voxel-based statistical analysis - not just of activation, but also of anatomy, as in voxel-based morphometry (VBM) - across a group, normalizing can largely remove a huge source of error from your data by removing variance between brain shapes.

The disadvantage is just as simple: Like any preprocessing, normalization isn't perfect. It generally requires interpolation, which introduces small errors into the images, and even with normalization, anatomies may not line up as well as you'd hope. It can also be slow - depending on the methods and programs used, normalizing a run of functional images can take hours or days. Still, to use voxel-based statistics, it's a necessary evil...

4. When is it unhelpful to normalize?
*********************
If you're running an analysis that's not voxel-based - say, one that's based on region-of-interest-specific timecourses - then normalization makes a lot less sense. An alternative to voxel-based methods is to compare some measure activation in particular structures hand-drawn (or automatically drawn) on individual subjects' images. Since a method like this gets summary statistics out from each subject individually, without requiring that any statistical images be laid on top of each other, normalization is totally unnecessary. Some researchers choose to preprocess their data on two parallel paths, one with normalization and one without, using the non-normalized data for region-of-interest analysis and the normalized for traditional voxel-based methods.

As well, several factors can make normalization difficult. Lesions, atrophy, different developmental stages, neurological disorders, and other problems can make standard normalization impossible. Some of these problems can be easily addressed (see Brett et. al), and some can't be. Anyone using patient populations with significant neurological differences from their normalization templates should be advised to explore the literature on normalizing patients before proceeding.

5. How important is it to align images to AC-PC before normalizing?
*********************
This varies between programs. For AFNI and BrainVoyager, it's pretty important. The nonlinear transformations can account for non-aligned images in theory, but if you start the images off in a non-aligned state, the algorithm is more likely to get caught in a local minimum of the search space, and give you strange normalization parameters. If you aren't realigning before normalizing, it's best to make sure to examine the normalized brains afterwards to make sure that your normalization ran okay. SPM's normalization algorithm has a realignment phase built in that runs automatically before the nonlinear transformations are examined, so doing realignment beforehand isn't necessary. It can't hurt, particularly when realigning runs of functional data, and it's still wise to examine the normalized image afterwards as a sanity check...

6. How important is it to make sure your segmentation is good before normalizing to the gray template?
*********************
Very important. The gray template contains, in theory, only gray-matter voxels. Normalization algorithms find their transformations by trying to minimize the voxel-by-voxel intensity differences between images, and white matter, CSF and gray matter all have notably different intensity profiles. So if you have left-over fringes of white matter or CSF or occasional speckles of white matter included by error in your gray-matter image, they'll be treated as error voxels even if they're in the right place. The algorithm may still converge to the best gray-matter solution, but you can greatly increase your chances of getting a good gray-matter normalization by making sure your segmentation is clean and only includes gray-matter voxels.

7. Should you use the inplane anatomy or the high-res anatomy to determine parameters?
*********************
There's not a perfect answer, but probably if you have a high-res anatomy, you should use it. In theory, the high-res anatomy should provide you a better match, because it has more detail. However, if you have significant head movement between the high-res scan and the functionals, there will be an additional source of error in the high-res (even after realignment) that may not be there in the inplane if there's less movement between the inplane and the functionals. In general, though, the increased resolution of the high-res will probably provide better precision for your normalization parameters. In practice, the difference will probably be small, but every little bit helps...

8. When in my analysis stream should I normalize?
*********************
There are two obvious points when you can normalize - a) in the individual subject analysis, before you estimate your model / do your stats, b) after you've done your stats and calculated contrast images for each subject, but before you do your group analysis. In case a), you'll normalize all your functional images (usually after estimating parameters from an anatomical image); in case b), you'll normalize only your contrast images (always after estimating parametes from an anatomical image). In general, the standard is a), but I'm not sure exactly why. One problem with b) might be that interpolation errors are being introduced directly into your summary statistics, rather than in the functional images they're derived from. To the extent that contrast images are less smooth than functional images, this will tend to disadvantage b). As well, those interpolation errors are then going to be averaged over far fewer observations in b) - when you're combining only one contrast image for each person - than in a) - when you're often combining several hundred functional images for each person. Not sure whether this will make much difference, though... This is a test that should be run.

9. How can you tell how good your normalization is?
*********************
There are possible automated ways you can use to determine quantifiably how close your normalization has gotten - see Salmond et. al, for one - but, in general, the easiest way to go is just to compare the template image and your normalized image by looking at them side-by-side (or overlaid). Check to make sure that the gross structures and lobes line up reasonably well, and if you have any particular area of interest - hippocampus, V1, etc. - check those in detail to make sure they line up okay. If they don't, you may want to align your source image differently before normalizing, or try normalizing just your gray matter.

9. What’s the difference between linear and nonlinear transformations?
*********************
Roughly, linear transformations are those that treat the head as a rigid body, and allow it to be transformed only in ways that don't affect its shape or the shape of anything inside it. Rotations, translations, and scaling all fall in this category. Nonlinear transformations are any transformations that don't respect those constraints; these include any transformations that squeeze parts of the image, stretch parts of the image, or generally distort the shape of the head in any way.

10. How do I normalize children’s brains? How about aging brains?
*********************
There is a monster literature out there on normalizing various non-standard brains, too large to survey easily; the Wilke et. al is a good start for children and contains some good citations into that literature, and the Brett et. al contains some similarly nice citations for aging brains. Anyone who knows this literature well is invited to contribute links and/or citations...


P threshold
===================

1. What is the multiple-comparison problem? What is familywise error correction (FWE)?
*********************
To start, Nichols and Hayasaka provide an excellent introduction to the issue of FWE in neuroimaging in very readable fashion. You're encouraged to check it out.

Many scientific fields have had to confront the problem of assessing statistical significance in the context of multiple tests. With a single statistical test, the standard conventionally dictates a statistic is significant if it is less than 5% likely to occur by chance - a p-threshold of 0.05. But in fields like DNA microassays or neuroimaging, many thousands of tests are done at once. Each voxel in the brain constitutes a separate test, which usually means tens of thousands of tests for a given subject. If the conventional p-threshold of 0.05 is applied on a voxelwise basis, then, just by chance you're almost guaranteed to have many hundreds of false-positive voxels. In order to avoid any false positives, then, researchers generally correct their p-threshold to account for how many tests they're performing. This type of correction prevents Type I error across the whole family of tests you're doing - a familwise error correction, or FWE correction.

The standard approach to FWE correction has been the Bonferroni correction - simply divide the desired p-threshold by the number of tests, and you'll maintain correct control over the FWE rate. In general, the Bonferroni correction is a pretty conservative correction, and it suffers from a fatal flaw with neuroimaging data. The Bonferroni correction demands that all the tests be independent from each other, and that demand is manifestly not fulfilled in neuroimaging data, where there is a complex, substantial and generally unknown structure of spatial correlations in the data. Essentially, the Bonferroni correction assumes there are more spatial 'degrees of freedom' than there really are; one voxel is not independent from the next, and so one only needs to correct for the 'true' number of independent tests you're doing. This effort, though, is tricky, and so a good deal of theory has been developed on ways around Bonferroni-type corrections that still control the FWE at a reasonable level.

2. What is Gaussian random-field theory and how does it apply to FWE?
*********************
Worsley et. al is one of the first papers to link random-field theory with neuroimaging data, and that link has been tremendously productive in the years since. Random-field theory (RFT) corrections attempt to control the FWE rate by assuming that the data follow certain specified patterns of spatial variance - that the distributions of statistics mimic a smoothly varying random field. RFT corrections work by calculating the smoothness of the data in a given statistic image and estimating how unlikely it is that voxels (or clusters or patterns) with particular statistic levels would appear by chance in data of that local smoothness. The big advantages of RFT corrections are that they adapt to the smoothness in the data - with highly correlated data, Bonferroni corrections are far too severe, but RFT corrections are much more liberal. RFT methods are also computationally extremely efficient.

However, RFT corrections make many assumptions about the data which render the methods somewhat less palatable. Chief among these is the assumption that the data must have a minimum level of smoothness in order to fit the theory - at least 2-3 times the voxel size is recommended at minimum, and more is better. For those researchers unwilling to pay the cost in resolution that smoothing imposes, RFT methods are problematic. As well, RFT corrections are only available for statistics whose distributions in a random field have been laboriously calculated and derived - the common statistics fall in this category (F, t, minimum t, etc.), but ad hoc statistics can't be corrected in this manner. Finally, it's become clear (and Nichols and Hayasaka), that even with the assumptions minimally satisfied, RFT corrections tend to be too conservative.

Random-field theory corrections are available by default in SPM; in SPM99 or earlier, choosing a "corrected" p-threshold means using an RFT correction, while in SPM2, choosing the "FWE" correction to your p-threshold uses these methods. I don't believe corrections of this sort are available in AFNI or BrainVoyager.

3. What is false discovery rate (FDR)? How is it different from other types of multiple-comparison correction?
*********************
RFT methods may have their flaws, but some researchers have pointed out a different problem with the whole concept of FWE correction. FWE correction in general controls the error rate for the whole family; it guarantees that there's only a 5% chance (for example) of any false positives appearing in the data. This type of correction simply doesn't fit the intuition of many neuroimaging researchers, because it suggests that every voxel activated is a true active voxel, and most researchers correctly assume there's enough noise in every stage of the process to make a few voxels here and there look active just by chance. Indeed, it's rarely of crucial interest in a particular study whether one particular voxel is necessarily truly or falsely positive - most researchers are willing to accept that some of their signal is actually noise - but that level of inference is precisely what FWE corrections attempt to license.

Benjamini & Hochberg, faced with this conundrum, developed a new idea. Rather than controlling the FWE rate, what if you could control the amount of false-positive data you had? They developed a method to control the false discovery rate, or FDR. Genovese et. al recently imported this method specifically into neuroimaging. The idea in controlling the FDR is not to guarantee you have no false positives - it's to guarantee you only have a few. Setting the FDR control level to 0.05 will guarantee that no more than 5% of your active voxels are false positives. You don't know which ones they might be, and you don't even know if fully 5% are false positive. But no more than 5% are falsely active.

The big advantage of FDR is that is adapts to the level of signal present in the data. With small signal, the correction is very liberal. With huge signal, it's relatively more severe. This adaptation renders it more sensitive than an RFT correction if there's any signal present in the data. It allows a much more liberal threshold to be set than RFT, at a cost that most researchers have already mentally paid - a few false positive voxels. It requires almost no computational effort, and doesn't require laborious derivations to be used with new statistics.

FDR is not a perfect cure-all - it does require some assumptions about the level of spatial correlation in the data. At the outer bound, allowing any arbitrary correlation structure, it is only slightly more liberal than the equivalent RFT correction. But with looser assumptions, it's a great deal more liberal. Genovese et. al have argued that fMRI data in many situations fits a very loose set of assumptions, enabling a pretty liberal correction.

The latest edition of every major neuroimaging program provides some methods for FDR control - SPM2 and BrainVoyager QX have it built-in, and AFNI's 3dFDR program does the same work. Tom Nichols has predicted FDR methods will essentially replace most FWE correction methods within a few years, and they are beginning to be widely used throughout neuroimaging literature.

4. What is permutation testing? How is it different from other types of multiple-comparison correction?
*********************
Permutation testing is a form of non-parametric testing, and Nichols and Holmes give an excellent introduction to the field in their paper, a much better treatment than I can give it here. But here's the extreme nutshell version. Permutation tests are a sensitive way of controlling FWE that make almost no assumptions about the data, and are related to the stats/CS concept of 'bootstrapping.'

The idea is this. You hope your experimental manipulation has had some effect on the data, and to the extent that it has, your design matrix is a model that explains the data pretty well, with large beta weights for the conditions of interest. But what if your design matrix had been different? What if you randomly re-labeled your trials, so that a trial that was actually an A trial in the real experiment was re-labeled as a B, and put into the design matrix as a B, and a B trial was re-labeled and modeled as a C trial, and a C as an A, and so forth. If your experiment had a big effect, the new, randomly mixed-up design matrix won't explain it well at all - if you re-ran your model using that matrix, you'd get much smaller beta weights. Of course, on the null hypothesis, there wasn't any effect at all due to your manipulation, which means the random design matrix should explain it just as well.

And now that you've re-labeled your design matrix and re-run your stats, you mix up the design matrix again, differently and do the same thing. And then do it again. And again, until you've run through all the possible permutations of the design matrix (or at least a lot of them). You'll end up with a distribution of beta weights for that condition from possible design matrices. And now you go back and look at the beta weight from your real experiment. If it's at the extreme end of that distribution you've created - congrats! You've got a significant effect for that condition. The idea in permutation testing is you don't make any assumptions about what the statistic distribution could be - you go out and empirically determine it, from your own real data.

But how does that help you with the multiple-comparison problem? One nice thing about permuation testing is that aren't restricted to testing significance for stats with known distributions, like t or F. We can use these on any ad hoc statistic we like. So let's do it across the design matrices, using as our statistic the maximal T: the value of the maximum T-statistic in the whole image for that design matrix. We come up with a distribution, just like before, and we can find the t-statistic that corresponds to the 5% most extreme parts of the maximal T distribution. And now, the clever bit: we go back to our real experiment's statistical map, and threshold it at that 5% level from the maximal T. Hopefully the t-statistics from our real experiment are generally so much higher than those from the random design matrices as to mean a lot of voxels in our real experiment will have t-statistics above that level - and we don't need to correct their significance at all, because anything in that extreme part of the maximal T distribution is guaranteed to be among the most extreme possible t-statistics for any voxel for any design matrix.

Permuation tests have the big advantages of making almost no (but not totally none - see Nichols and Holmes for details) assumptions about your data, which means they work particularly well with low degrees of freedom, where other methods' assumptions about the shape of their statistic's distribution can be violated. They also are extremely flexible - any true or ad hoc statistic can be tested, such as maximal T, or size of structure, or voxel's favorite color - anything. But they have a big disadvantage: computational cost. Running a permutation test involves re-estimating at least 20 models to be able to guarantee a 0.05 significance level, and so in SPM for individual data, that cost can be prohibitive. For other programs, the situation's not as bad, but it can still be pretty difficult to wait. Permuation tests are available at least in SPM99 with the SnPM toolbox, and in AFNI with the 3dMonteCarlo program. Not sure about BrainVoyager.

5. When should I use different types of multiple-comparison correction?
*********************
Nichols and Hayasaka's paper does an explicit review of various FWE correction methods (as well as FDR) on simulated and real data of a variety of smoothness levels and degrees of freedom, to judge how conservative or liberal different methods were. Their main findings are:

* Random-field corrections are extremely conservative for all smoothnesses except the highest. This bias becomes stronger as the degrees of freedom go down, such that low-degree-of-freedom, low-smoothness images corrected with RFT methods show the worst underactivation. At the highest smoothness (8-12mm FWHM), they perform reasonably well for all df.
* Permutation methods are almost exact for all degrees of freedom and for all smoothnesses. They become slightly better with data of high smoothness, but basically perform tremendously well under all conditions.
* FDR is not strictly speaking intended to control FWE, but it does an excellent job doing so for low-smoothness data at all degrees of freedom. At high smoothnesses (6mm FWHM and greater), the correction becomes too conservative.

Accordingly, the nutshell recommendations are as follows:

* Random-field methods are good for highly-smoothed data only and are best for single-subject data. For researchers who need a good deal of smoothing to collect significant signal, or who aren't particularly interested in very fine resolution, RFT corrections are quite exact and easily implemented for single subjects. At low degrees of freedom for any smoothness (say, less than 20 df), the RF corrections are generally too conservative for any smoothness.
* For unsmoothed (or low-smoothed), single-subject data, FDR corrections are the best. They have very high sensitivity while still providing good control of false positives, even with low degrees of freedom. Group data tend naturally to be smoother than single-subject data, due to the blurring imposed by anatomical variability, and so may not be ideal for FDR corrections.
* Permutation tests are optimized for group data - they perform perfectly at very low degrees of freedom, where other methods' assumptions are invalidated, and they improve slightly with high-smoothness data, although they still do fine with unsmoothed. In group testing, the permutation is whether each subject's t-statistic signs are true or flipped - presumably, if the mean is zero, flipping the sign of the statistic won't make a difference, but if the mean is nonzero, that flipping will matter. As well, the relative speed of estimating group models in most programs helps counter the increased computational cost of permutation testing in general.

6. What is small-volume correction?
*********************
All the FWE correction methods here adapt to the number of tests performed. The fewer tests, the less severe the correction, and in neuroimaging, the number of tests performed corresponds to the number of voxels or the volume corrected. So it's to your advantage when doing FWE correction to minimize the volume you're testing. If you have an a priori hypothesis about where you might see activation, like a particular anatomical structure or a particular area found to be active in another study, you might restrict your correction to only that area and be perfectly valid in only performing FWER correction there. In practice, this is often done when a particular activation is above the uncorrected threshold, but you'd like to report corrected statistics. You might also try it when you're using a corrected threshold to start, but not seeing any activation where you might expect some - you could restrict your correction to a smaller volume than the whole brain and suddenly get activation popping up above the new, small-volume-corrected threshold.

SPM has a shortcut to this sort of volume restriction - the small volume correction (or S.V.C.) button in the results interface. It'll let you re-calculate corrected p-statistics for a specified region only - an ROI mask image, or a sphere around a point, etc. This change won't change the uncorrected p-statistics for any activations, but it will make the corrected p-statistic for any activations in that region significantly better, depending on how big your specified region is.

Note that if you're using an uncorrected threshold to start, using S.V.C. won't show you anything new. This correction only re-jiggers the corrected p-statistic for a given region.

7. What do all the different reported values in my SPM table mean (p-corrected, p-uncorrected, cluster, set, etc.)? How are they calculated?
*********************
SPM reports a pair of p-statistics for each voxel, a p-statistic for each cluster, and a p-statistic for each set. At the voxel level, these are relatively self-explanatory. The p-uncorrected statistic is the probability that, by itself, a voxel with that t- (or F-)statistic would occur just by chance. This is the statistic that's used to threshold the brain using the uncorrected threshold, or "None" correction in SPM2. The p-corrected statistic is the probability of that same t-statistic, but corrected for FWE using Gaussian RFT methods. This statistic reflects the volume that's being corrected (and hence changes in small-volume-corrected regions).

The cluster and set values are more obscure and less useful - they're explained in detail in the Friston et. al. Briefly, the cluster-level p-statistic is the probability that a cluster of that size would occur just by chance in data of the given smoothness. The key difference is that the activation of a cluster doesn't imply that any particular voxel in the cluster is active - you can't use that statistic to license inference that any one voxel in the cluster is above some threshold. The set-level p-statistic is similar, at the level of the whole brain; it's the probability that a pattern of activation of that size (number of clusters) would occur in data of the given smoothness. But it doesn't mean that any given cluster is active - it only tells you that there's some particular pattern of activation happening, in a regionally unspecific manner. Because both of these statistics are derived from Gaussian RFT theory, they're both, by definition, corrected p-statistics. But because neither of them license inference to any particular voxel, they're not widely used or cited.

8. What should my p-threshold be for some analysis X?
*********************
p < 0.05, corrected, remains the gold standard for any neuroimaging analysis. Because RFT corrections are so severe, though (and because other methods aren't widespread enough to challenge them), a de facto standard of p < 0.001 seems to be in operation these days a lot of the time. Depending on the type of analysis, you may be able to go even looser - group-level regressions are sometimes seen more loosely, such as p < 0.005, although there's not a particularly good reason for this.

Using FDR control instead of FWE correction is relatively new, so by default an FDR of 0.05 seems to be the current standard, but Benjamini & Hochberg, among others, have argued that a more liberal threshold in some situations may be reasonable - as high as 0.1 or even a bit higher.

For any type of non-voxel-based analysis, such as correlations of beta weights, etc., p < 0.05 is still the magic number for most reviewers.

9. What should my p-threshold be for conjunction analyses?
*********************
A good question. Check out the conjunction papers for more detail, but the basic argument is simple. If a voxel that's active in a conjunction analysis simply has to be active in all of the component analyses, and you're thresholding the conjunction (not any component analyses), then the component analyses should have lower thresholds than the conjunction. Specifically, if you wanted to threshold the conjunction at p < 0.001, and you had two components to the conjunction, then you should threshold each of the components at sqrt(0.001). Any voxel active in both of those at that level will be less likely in the conjunction, so you can threshold each component at a very liberal level and be sure the conjunction's threshold will be quite stringent. In short - the conjunction threshold is the product of the component thresholds.

Many researchers, however, disagree with this line of reasoning. First, obviously, this argument depends on all of the components being independent - if they're dependent at all, then the product of the individual thresholds will be more stringent than the true conjunction threshold. Even if they're all independent, though, it's clear that using this line of argument means that any active voxel in the conjunction is a voxel that may well not be active at a "reasonable" threshold in any of the components. This problem is exacerbated with more than two components - with three, say, each component could be thresholded at p < 0.1 uncorrected, and the conjunction could have a threshold of p < 0.001. This flies in the face of what many people try to argue about their conjunctions, which is that they represent areas that are activated in all of their components. So many researchers use the strategy of simply thresholding their individual components at some liberal but reasonable threshold - p < 0.001, or p < 0.005 - and then simply assess the intersection of the active areas as the conjunction. This clearly results in extremely significant p-statistics in the conjunction, but it at least gets closer to the idea of "conjunction" that most researchers seem to have.

10. What should my p-threshold be for masked analyses?
*********************
If you're masking your one analysis with the results of an another analysis, you're basically doing a conjunction (see above), so you can liberalize your threshold at least a bit. If you're masking your analysis with a region of interest mask, anatomical or otherwise, you might also consider using a small volume correction and using p < 0.05 corrected as a threshold. If you're doing some other crazy kind of mask... well, you're kind of in uncharted waters. Start with something reasonable and go from there, and good luck to you.


Percent Signal Change
===================

Check out `ROI`_ for more info about region-of-interest analysis in general...

1. What’s the point of looking at percent signal change? When is it helpful to do that?
*********************
The original statistical analyses of functional MRI data, going way back to '93 or so, were based exclusively on intensity changes. It was clear from the beginning of fMRI studies that raw intensity numbers wouldn't be directly comparable across scanners or subjects or even sessions - average means of each of those things varies widely and arbitrarily. But simply looking at how much the intensity in a given voxel or region jumped in one condition relative to some baseline seemed like a good way to look at how big the effect of the condition was. So early block experiments relied on averaging intensity values for a given voxel in the experimental blocks, doing the same for the baseline block, and comparing the two of 'em. Relatively quickly, fancier forms of analysis became available, and it seemed obvious that correcting that effect size by its variance was a more sensitive analysis than looking at it raw - and so t-statistics came into use, and the general linear model, and so forth.

So why go back to percent signal change? For block experiments, there are a couple reasons, but basically percent signal change serves the same function as beta weights might (see `ROI`_ for more on them): a numerical measure of the effect size. Percent signal change is a lot more intuitive a concept than parameter weights are, which is nice, and many people feel that looking at a raw percent signal change can get you closer to the data than looking at some statistical measure filtered through many layers of temporal preprocessing and statistical evaluation.

For event-related experiments, though, there's a more obvious advantage: time-locked averaging. Analyzing data in terms of single events allows you to create the timecourse of the average response to a single event in a given voxel over the whole experiment - and timecourses can potentially tell you something completely different than beta weights or contrasts can. The standard general linear model approach to activation assumes a shape for the hemodynamic response, and tests to see how well the data fit that model, but using percent signal change as a measure lets you actually go and see the shape of the HRF for given conditions. This can potentially give you all kinds of new information. Two voxels might both be identified as "active" by the GLM analysis, but one might have an onset two seconds before the next. Or one might have a tall, skinny HRF and one might have a short but wide HRF. That sort of information may shed new light on what sort of processing different areas are engaging in. Percent signal change timecourses in general also allow you to validate your assumptions about the HRF, correlate timecourses from one region with those from another, etc. And, of course, the same argument about percent signal change being somehow "closer" to the data still applies.

Timecourses are rarely calculated for block-related experiments, as it's not always clear what you'd expect to see, but for event-related experiments, they're fast becoming an essential element of a study.

2. How do I find it?
*********************
Good question, and very platform dependent. In AFNI and BrainVoyager, whole-experiment timecourses are easily found by clicking around, and in the Gablab the same is available for SPM with the Timeseries Explorer. Peristimulus timecourses, though, ususally require some calculation. In SPM, you can get fitted responses through the usual results panel, using the plot command, but those are in arbitrary units and often heavily smoothed relative to the real data. The simplest way these days for SPM99 is to use the Gablab Toolbox's roi_percent code. Check out RoiPercent for info about that function. That creates timecourses averaged over an ROI for every condition in your experiment, with a variety of temporal preprocessing and baseline options. In SPM2, the new Gablab roi_deconvolve is sort of working, although it's going to be heavily updated in coming months. It's based off AFNI's 3dDeconvolve function, which is the newest way to get peristimulus timecourses in AFNI. That's based on a finite impulse response (FIR) model (more on those below). BrainVoyager's ROI calculations will also automatically run an FIR model across the ROI for you.

3. How do those timecourse programs work?
*********************
The simplest way to find percent signal change is perfectly good for some types of experiments. The basic steps are as follows:

* Extract a timecourse for the whole experiment for your given voxel (or extract the average timecourse for a region).
* Choose a baseline (more on that below) that you'll be measuring percent signal change from. Popular choices are "the mean of the whole timecourse" or "the mean of the baseline condition."
* Divide every timepoint's intensity value by the baseline, multiply by 100, and subtract 100, to give you a whole-experiment timecourse in percent signal change.
* For each condition C, start at the onset of each C trial. Average the percent signal change values for all the onsets of C trials together.
* Do the same thing for the timepoint after the onset of each C trial, e.g., average together the onset + 1 timepoint for all C trials.
* Repeat for each timepoint out from the onset of the trial, out to around 30 seconds or however long an HRF you want to look at.

You'll end up with an average peristimulus timecourse for each condition, and even a timecourse of standard deviations/confidence intervals if you like - enough to put confidence bars on your average timecourse estimate. This is the basic method, and it's perfect for long event-related experiments - where the inter-trial interval is at least as long as the HRF you want to estimate, so every experimental timepoint is included in one and only one average timecourse.

This method breaks down, though, with short ISIs - and those are most experiments these days, since rapid event-related designs are hugely more efficient than long event-related designs. If one trial onsets before the response of the last one has faded away, then how do you know how much of the timepoint's intensity is due to the previous trial and how much due to the current trial? The simple method will result in timecourses that have the contributions of several trials (probably of different trial types) averaged in, and that's not what you want. Ideally, you'd like to be able to run trials with very short ISIs, but come up with peristimulus timecourses showing what a particular trial's response would have been had it happened in isolation. You need to be able to deconvolve the various contributions of the different trial types and separate them into their component pieces.

Fortunately, that's just what AFNI's 3dDeconvolve, BrainVoyager QX, and the Gablab's roi_deconvolve all do. SPM2 also allows it directly in model estimation, and Russ Poldrack's toolbox allows it to some degree, I believe. They all use basically the same tool - the finite impulse response model.

4. What's a finite impulse response model?
*********************
Funny you should ask. The FIR model is a modification of the standard GLM which is designed precisely to deconvolve different conditions' peristimulus timecourses from each other. The main modification from the standard GLM is that instead of having one column for each effect, you have as many columns as you want timepoints in your peristimulus timecourse. If you want a 30-second timecourse and have a 3-second TR, you'd have 10 columns for each condition. Instead of having a single model of activity over time in one column, such as a boxcar convolved with a canonical HRF, or a canonical HRF by itself, each column represents one timepoint in the peristimulus timecourse. So the first column for each condition codes for the onset of each trial; it has a single 1 at each TR that condition has a trial onset, and zeros elsewhere. The second column for each condition codes for the onset + 1 point for each trial; it has a single 1 at each TR that's right after a trial onset, and zeros elsewhere. The third column codes in the same way for the onset + 2 timepoint for each trial; it has a single 1 at each TR that's two after a trial onset, and zeros elsewhere. Each column is filled out appropriately in the same fashion.

With this very wide design matrix, one then runs a standard GLM in the multiple regression style. Given enough timepoints and a properly randomized design, the design matrix then assigns beta weights to each column in the standard way - but these beta weights each represent activity at a certain temporal point following a trial onset. So for each condition, the first column tells you the effect size at the onset of a trial, the second column tells you the effect size one TR after the onset, the third columns tells you the effect size two TRs after the onset, and so on. This clearly translates directly into a peristimulus timecourse - simply plot each column's beta weight against time for a given condition, and voila! A nice-looking timecourse.

FIR models rely crucially on the assumption that overlapping HRFs add up in linear fashion, an assumption which seems valid for most tested areas and for most inter-trial intervals down to about 1 sec or so. These timecourses can have arbitrary units if they're used to regress on regular intensity data, but if you convert your voxel timecourses into percent signal change before they're input to the FIR model, then the peristimulus timecourses you get out will be in percent signal change units. That's the tack taken by the Gablab new roi_percent. Some researchers have chosen to ignore the issue and simply report the arbitrary intensity units for their timecourses.

By default, FIR models include some kind of baseline model - usually just a constant for a given session and a linear trend. That corresponds to choosing a baseline for the percent signal change of simply the session mean (and removing any linear trend). Most deconvolution programs include the option, though, to add other columns to the baseline model, so you could choose the mean of a given condition as your baseline.

There are a lot of other issues in FIR model creation - check out the AFNI 3dDeconvolve model for the basics and more.

5. What are temporal basis function models? How do they fit in?
*********************
Basis function models are a sort of transition step, representing the continuum between the standard, canonical-HRF, GLM analysis, and the unconstrained FIR model analysis. The standard analysis assumes an exact form for the HRF you're looking for; the FIR places no constraints at all on the HRF you get. But sometimes it's nice to have some kinds of constraints, because it's possible (and often happens) that the unconstrained FIR will converge on a solution that doesn't "look" anything like an HRF. So maybe you'd like to introduce certain constraints on the type of HRFs you'll accept. You can do that by collapsing the design matrix from the FIR a little bit, so each column models a certain constrained fragment of the HRF you'd like to look for - say, a particular upslope, or a particular frequency signature. Then the beta weight from the basis function model represents the effect size of that part of the HRF, and you can multiply the fragment by the beta weight and sum all the fragments from one condition to make a nice smooth-looking (hopefully) HRF.

Basis function models are pretty endlessly complicated, and the interested reader is referred to the papers by Friston, Poline, etc. on the topic - check out the Friston et. al, "Event-related fMRI" paper.

6. How do you select a baseline for your timecourse? What are pros and cons of possible options? Do some choices make particular comparisons easier or harder?
*********************
Good question. Choosing a particular baseline places a variety of constraints on the shape of possible HRFs you'll see. The most popular option is usually to simply take the mean intensity of the whole timecourse - the session mean. The problem with that as a baseline is that you're necessitating that there'll be as much percent signal change under the baseline as over it. If activity is at its lowest point during the inter-trial interval or just before trial onset, then, that may lead to some funny effects, like the onset of a trial starting below baseline, and dramatic undershoots. As well, if you've insufficiently accounted for drifts or slow noise across your timecourse, you may overweight some parts of the session at the expense of others, depending on what shape the drift has. Alternatively, you could choose to have the mean intensity during a certain condition be the baseline. This is great if you're quite confident there's not much response happening during that condition, but if you're not, be careful. Choosing another condition as the baseline essentially calculates what the peristimulus timecourse of change is between the two conditions, and if there's more response at some voxels than you thought in the baseline condition, you may seriously underestimate real activations. Even if you pick up a real difference between them, the difference may not look anything like an HRF - it may be constant, or gradually increase over the whole 30 seconds of timecourse. If you're interested in a particular difference between two conditions, this is a great option; if you're interested in seeing the shape of one condition's HRF in isolation, it's iffier.

With long event-related experiments, one natural choice is the mean intensity in the few seconds before a trial onset - to evaluate each trial against its own local baseline. With short ISIs, though, the response from the previous trial may not have decayed enough to show a good clean HRF.

7. What kind of filtering should I do on my timecourses?
*********************
Generally, percent signal analysis is subject to the same constraints in fMRI noise as the standard GLM, and so it makes sense to apply much of the same temporal filtering to percent signal analysis. At the very least, for multi-session experiments, scaling each session to the same mean is a must, to allow different sessions to be averaged together. Linear detrending (or the inclusion of a first-order polynomial in the baseline model, for the AFNI users) is also uncontroversial and highly recommended. Above that, high-pass filtering can help remove the low-frequency noise endemic to fMRI and is highly-recommended - this would correspond to higher-order polynomials in the baseline model for AFNI, although studies have shown anything above a quadratic isn't super useful (Skudlarski et. al). Low-pass filtering can smooth out your peristimulus timecourses, but can also severely flatten out their peaks, and has fallen out of favor in standard GLM modeling; it's not recommended. Depending on your timecourse, outlier removal may make sense - trimming the extreme outliers in your timecourse that might be due to movement artifacts.

8. How can you compare time courses across ROIs? Across conditions? Across subjects? (peak amplitude? time to peak? time to baseline? area under curve?) How do I tell whether two timecourses are significantly different? How can you combine several subjects’ ROI timecourses into an average? What’s the best way?
*********************
All of these are great questions, and unfortunately, they're generally open in the literature. FIR models generally allow contrasts to be built just as in standard GLM analysis, so you can easily do t- or F-tests between particular aspects of an HRF or combinations thereof. But what aspects make sense to test? The peak value? The width? The area under the curve? Most of these questions aren't super clear, although Miezin et. al and others have offered interesting commentary on which parameters might be the most appropriate to test. Peak amplitude is the de facto standard, but faced with questions like whether the tall/skinny HRF is "more" active than the short/fat HRF, we'll need a more sophisticated understanding to make sense of the tests.

As for group analysis of timecourses, that's another area where the literature hasn't pushed very far. A simple average of all subjects' condition A, for example, vs. all subjects' condition B may well miss a subject-by-subject effect because of differing peaks and shapes of HRFs. That simple average is certainly the most widely used method, however, and so fancier methods may need some justification. One fairly uncontroversial method might be simply analogous to the standard group analysis for regular design matrices - simply testing the distribution across subjects of the beta weight of a given peristimulus timepoint, for example, or testing a given contrast of beta weights across subjects.


Physiology and fMRI
===================

This section is intended to address design-related questions that focus primarily on how physiological factors can affect your scanning - things like heart rate, breathing, the types of artifacts generated by those movements, etc. Obviously, physiological factors are mixed in heavily with your experimental design, so be sure to check out some other design-related pages:

* `Design`_
* `Scanning`_
* `Jitter`_

1. Why would I want to collect physiological data?
*********************
No, the real question should be: why wouldn't you want to collect physiological data? And the only answer is: because you hate freedom. Haha! Phew.

Actually, the main reason is because physiological effects can be a significant source of noise in your data. The pulsing of blood vessels with the cardiac cycle can move brain tissue, jostle ventricles and alter BOLD signal in specific regions; respiration can induce magnetic inhomogeneities and create head movements. If you measure physiology, you can, at least in part, account for those sources of noise and remove their confounding effects from your data, boosting your signal to noise ratio.

Secondarily (well, primarily, for some), for many studies, physiological measures like heart rate or respiration rate can be an important source of data themselves, providing an important test of autonomic arousal that may supplement self-report data or other measures. All those uses are beyond the scope of this discussion, but it's something to keep in mind.

2. How do I do it?
*********************
Thankfully, scanner designers have generally had the foresight to think someone might want to collect this info, and so most scanners have instruments built in to record at least two main physiological measures - heartrate/cardiac cycle (usually with a photoplethysmograph - a small clip that goes on the finger) and respiration (usually with a pneumatic belt - a thin belt that wraps around the chest). For scanners without these instruments built in, several companies manufacture MR-compatible versions of these instruments.

There may be other measures that you may want to collect - galvanic skin response, for example - and here at Stanford, those sensors are relatively easily available. Other measures have less of an effect on creating noise in the signal, however, and so we'll primarily address those two measures.

3. How might the cardiac cycle influence my signal?
*********************
Several ways. The pulsation of vessels creates a variety of other movements, as CSF pulses along with it to make room for incoming blood, tissue moves aside slightly as vessels swell and shrink, and waves of blood (and the accompanying BOLD signal) move through the head. This sounds like the effects can be relatively small, but large structures of the brain can move significant amounts in the neighborhood of large vessels, and the resulting motion can significantly change your signal. Generally, the term "pulsatility" is used to describe the process that generates artifacts through cardiac movement. As well, because TRs for many experiments are slower than the cardiac cycle, these effects can occur image-to-image in an unpredictable way, as various points in the cycle are sampled in an irregular fashion.

These effects are more significant in some regions of the brain than in others; Dagli et. al address the question of where pulsatility artifact is worst. Perhaps unsurprisingly, areas near large vessels tend to rank among those regions, but other areas are also affected - regions near the borders of are also significantly affected.

4. How might respiration influence my signal?
*********************
Also a couple ways. Respiration, by its nature, can cause the head and particular parts of it (sinus cavities, etc.) to move slightly, which can induce motion-related changes in signal. Perhaps more significantly, the inflating and deflating of the lungs changes the magnetic signature of the human body, and that signature change can induce inhomogeneities in the baseline magnetic field (B0) that you've carefully tuned with your shim. Those inhomogeneities can be unpredictable and can affect your signal in unpredictable ways. Breathing rate can also be significantly less predictable than cardiac cycle - many subjects take spontaneous deeper breaths at irregular intervals, for example. Van de Moortele et. al address the sources of respiration artifact in some detail.

5. What can I do to account for these changes?
*********************
Thought you'd never ask. There are several ways. Pfeuffer et. al present a navigator-based method in k-space that adjusts, in large part, for global effects, more due to respiration changes. Perhaps the most prevalent ways, though, are retrospective, and rest on the fact that generally, cardiac cycle and respiration cycle take place at a time scale far faster than stimulus presentation for most experiments. Isolating signal changes that take place at the appropriate frequency, then, can help isolate those sources of noise.

This isn't as easy as it sounds, due to the potential aliasing of this noise because of the difference between TR and cardiac cycle time. But it is possible, with some work. Glover et. al present one of the industry-standard ways of doing this correction - by sorting images according to their point in the cardiac or respiration cycle, the appropriate amount of signal change due to those sources can be identified and removed from the image. This correction happens at the point of reconstruction of images from raw data, and is available automatically at Stanford in the makevols program.

Care should be taken with this option, though. As with any type of algorithm that removes "confounding" signals (realignment, say), this correction can't account for the extent to which physiological noise is correlated with the task. If there is significant correlation between your task and your physiological measures, removing noise due to the physiological sources will also remove task-related signal. This could happen with rapid event-related tasks if subjects breath in time with the tasks, or in any sort of experiment that might induce arousal - emotionally arousing stimuli may increase heart rate and respiration rate and thus change those noise profiles in a task-correlated way. In this case, Glover et. al suggest using only resting-state images to calculate this correction; this is always a significant consideration in physiological artifacts.

6. Are there other sources of physiological noise I might want to worry about?
*********************
Possibly. Peeters and Van der Linden address the question of longer-term physiological changes, such as those induced by pharmacological manipulations or sudden environmental changes. As a drug begins to be absorbed, for example, there can be a gradual change in vasoconstriction or blood oxygenation in general that can look like a global signal drift but is, in fact, a changing of the sources of the signal. Researchers interested in looking at these sorts of changes should look with care at these sorts of corrections.


ROI
===================

1. What's the point of region-of-interest (ROI) analysis? Why not just use the basic voxelwise stats? Are you too good for them or something? Huh, Mr. Fancy Pants?
*********************
Whoa, now, no need to get touchy. A lot of people think ROI analysis is a really good idea, possibly where the real future of fMRI lies. Nieto-Castanon et. al make the argument kind of like this: brain imaging is concerned, among other things, with analyzing how mental functions are connected to brain anatomy. Note that the word "voxel" didn't enter into that statement. Voxels aren't, on their own, a particularly useful concept for us, but the standard statistical model does all of its preprocessing and analysis on that level. That analysis path tends to completely blur anatomical boundaries, often by a great deal, smearing our resolution to hell and preventing us from making good clean associations between structure and function. So ROI analysis offers us a way to get around individual anatomical variability and sharpen our inferences.

As well, even if we don't start our statistical analysis at the ROI level, ROIs offer us a reasoned way to extract measures that differ from the standard voxelwise t-statistic. Measures like percent signal change timecourses or fit coefficients are additional information that you can extract from your data only by looking within a particular region. These measures can shed light on otherwise obscured aspects of your study - temporal characteristics, particular sizes and directions of effects, or correlations with behavior. Seen from this perspective, ROI analysis is a valuable parallel tool to the standard voxelwise GLM analysis and can often provide new and interesting pieces of the puzzle of your data.

2. How should I generate ROIs? What are the pros and cons of each way?
*********************
Funny you should ask; I just happened to have this little grid lying around, which has been helpfully converted into a sort of tree for the Digi-Web. The methods of ROI definition can be split along two axes - the type of brain ROIs are defined on (individual, group, atlas) and the features used to define it (microanatomy/cytoarchitecture, macroanatomy, function). The breakdown goes like this:

**Microanatomy / cytoarchitecture ROIs**

* Individually-defined: This would mean drawing (by hand or automated method) ROIs on the cytoarchitectonic map for an individual. The pros of this method: you're quite close to the neuronal level of organization, you've got very high resolution on your ROI, and the mapping between cytoarchitectonic regions has generally proven to be close (see Brett et. al). Cons: Not all functional info is represented (columnar structure, for example, is sometimes represented in cytoarchitecture and sometimes not), and, of course, getting the cytoarchitectonic map for a living human non-invasively is practically impossible at this point. That may change in the future, though - some groups are attempting to mine this data from fMRI maps.
* Group-defined: This is almost never done - it would mean drawing cytoarchitectonic maps on a template or something.
* Atlas-defined: This means getting coordinates for particular cytoarchitectonic regions from an atlas, and generally this means using the Talairach Daemon or some similar tool to get Brodmann Areas for particular atlas coordinates. Pros of this: the Tal Daemon's easy to get, easy to use, fast, popular, and can be easily used for MNI-space coordinates with a simple transform. Cons: Lots of sources of error in labeling. The original Talairach BA labeling is very crude (see Brett et. al) and based on eyeball. The Talairach - MNI transform isn't perfect. And perhaps biggest of all, there is enormouse individual variability of the shape and size of cytoarchitectonic regions - what's BA32 on the atlas may well be deep into BA10 on your subject. Some of those problems are solved by getting a better cyto. atlas - Zilles et. al are working on one - but the problem of individual variability remains.

**Macroanatomy / anatomically-defined ROIs**

* Individually-defined: This means drawing ROIs on each individual subject's anatomy, based on anatomical markers like sulci, gyri, and other features viewable by the naked eye. This drawing can be by hand or by an automated or semi-automated program . (See `Segmentation`_ for more info on those - they're getting pretty good.) Pros: Gets around problem of individual anatomical variability - by extracting each measure from an individually defined structure, you eliminate the issue of whether everyone's structures line up. Allows you to keep your data unsmoothed (smoothing is often done to avoid problems of anatomical variability), and so gives you higher resolution without sacrificing power and even while gaining some. Better mapping of function to structure. Cons: Can be hard to get - requires either good use of an automated algorithm or a lot of labor and experience in hand-drawing ROIs. Makes it more difficult to report activation coordinates (but note Swallow et. al suggest you can normalize your data before defining and be okay). Most importantly, the relationship between function and macroanatomy is very unclear in many regions of the brain, particularly associational cortex and prefrontal areas. Just because you're extracting from everybody's superior frontal gyrus doesn't mean you're going to get at all the same function. So this path may not gain you any power at all and might lose you some.
* Group-defined: Again, this is rarely done - it would mean drawing your ROI on some group average. Probably all of the cons of above, with fewer pros.
* Atlas-defined: This means taking some atlas system's description of anatomical features, like using the Talairach-defined amygdala or Talairach superior frontal gyrus, or possibly drawing an ROI on the MNI template brain. Often the ROIs are then reverse-normalized to fit the subject's non-normalized functional data. Pros: Very easy to get - probably the most-used method for defining anatomical ROIs. Very standardized, and so easily comparable across studies. There's not as much an issue with labeling as above - the Talairach and Tournoux atlas is very accurate at picking out coordinates for macroanatomical features, so you get to leverage their skill at drawing ROIs for your own study. Cons: The overlap of the atlas-defined region with your subjects' is only as good as your normalization - and Nieto-Castanon et. al offer a scathing commentary on how good standard normalization algorithms are, showing that even post-normalization, they had huge variations in the shape of particular gyri. Because of that, atlas-defined regions will always tend to obscure differences in anatomical variability, even with reverse normalization. As well, this method is still subject to the problem of unclear structure-function relationships in many areas of the brain.

**Functional ROIs**

* Individually-defined: This means choosing functionally-activated voxels from each individual's results - either from a localizer task or from the actual study task. This is so popular these days we've actually broken it out to its own page, FunctionalLocalization.
* Group-defined: This means choosing functionally-activated voxels from your group results and using that voxel set as an ROI to extract from individuals. Pros: Gets to function, as above. Can be a better guarantee that you've got some effect in a given region, since presumably voxels activated in the group had some decent activations in most or all subjects. A good chunk less labor than individually-defining ROIs. Cons: Swallow et. al demonstrate these ROIs are not as reliable (i.e., are more variable) than the individually-created ones. This method doesn't account at all for individual functional variability, which can be considerable (i.e., different voxels can be included or exluded wrongly in every subject).
* Atlas-defined: Probably the closest thing to this is using some other study's reported sites of functional activation - choosing ROIs that correspond to the Tal or MNI coords of other studies' activations. Pros: Avoids the problem of defining localizers, allows direct comparison between studies. Cons: Ignores differences in subject pool and anatomical and functional variability in your own subjects. Also introduces whatever error the other study might have had in localization into your study - if they got their spot wrong, you might be even wrong-er...
Lotta options. All of 'em have their pros and cons - which one you choose will depend largely on the type of questions you want to ask.

3. When can I just look at peak voxels vs. whole regions?
*********************
This is still an open question in the literature. The argument for averaging across an ROI is that it should enhance signal to noise; the timecourse from a single voxel can be quite noisy and could, indeed, be some kind of outlier in the ROI. Averaging might give you a better picture of what's happening over the whole ROI. The argument for using a peak voxel, though, is that we know the peak voxel - the voxel that shows the most correlation to the task relative to its variance - is guaranteed to show the best effect of any voxel in the ROI. Additionally, since we know our resolution is blurred by the vascular structure in the region, any spatial smoothing we may have done, and registration and normalization errors, it's entirely possible that some of our ROI's activation isn't reflecting "true" neuronal activity but simply an echo or blurring of activity elsewhere in the ROI. So to average those timecourse together may well wash out our effect, which is after all calculated in voxelwise fashion.

Nieto-Castanon et. al choose to look at whole ROIs, and that's arguably the prevailing sentiment in the literature. Particularly with false discovery rate p-threshold correction rising in prominence (see `P threshold`_), the risk of any given voxel being a false positive might seem too high.

On the other hand, at least one (and possibly more than one) empirical study - Arthurs & Boniface - has found that peak-voxel activity correlates better with evoked scalp electrical potentials than does activity averaged across an ROI. They cite a couple other studies that have examined similar issues in animal models, and suggest that in mammal cortex in general, the brain may "water the garden for the sake of one thirsty flower," i.e., ROI activity may only reflect true neuronal changes in a few voxels of the ROI. So the question remains open...

4. What sort of measures can I get out of ROIs?
*********************
The two big ones that are usually looked at are:

* beta weights (also called parameter weights or fit coefficients), which are voxel-by-voxel slope values from the multiple regression of your statistical model and correspond with effect sizes of particular conditions.
* percent signal change or timecourse information - literally looking at the TR-by-TR image intensity at a particular voxel or ROI to get a timecourse of intensities in a particular area. That timecourse is often trial-averaged to come up with time-locked average timecourse, which correspond to the average intensity change following the onset of a particular trial type - essentially an empirical look at the shape of the hemodynamic response to different trial types in a given region.

Other measures are occasionally taken - for voxel-based morphometry, for example, where you might look at percentage of gray matter in a given voxel - but those two are the biggies. Beta weights are used in block-related and event-related experiments; percent signal change is usually more important for event-related experiments, although it's occasionally used for blocks as well.

5. What’s the point of looking at percent signal? When is it helpful to do that? How do I find it?
*********************
For everything you could want to know about percent signal change, check out `Percent Signal Change`_.

6. What are beta weights / parameter weights / fit coefficients? When is it helpful to look at them? What types of analyses can I do with them?
*********************
When you run a general linear model to estimate your effect sizes (see `Basic Statistical Modeling`_ for info on this), you're essentially running a giant multiple regression on your data, with the columns of your design matrix as the regressors. Each of those columns corresponds to a particular effect, and each of them is assigned by the GLM a particular parameter value: the B in the equation Y = XB + E, where Y is the signal, X is the design matrix, and E is error. That parameter value corresponds to how large an effect the particular condition had in influencing brain activity.

Importantly, beta weights are not an index of how well your condition's design matrix fit the brain activity - it is not the r or r-squared value for the regression. It's the slope of the regression. This means you could conceivably have a very small effect that fit the model incredibly well, or a very large effect with a great deal of noise in the response. This slope corresponds better to the idea of 'level of activation for a particular condition' that we want to find. As an example, a design matrix column that was all zeros might predict the brain activity perfectly - there might be essentially no change in a given voxel down the whole timecourse. In that case, our r-squared would be very high for that column of the regression - but we wouldn't want to say that voxel was active, because it was totally insensitive to any experimental manipulation. It makes more sense to look at how big the effect size was - whether a given voxel seemed to respond very highly to a given trial type - and then, if we're concerned about noise, we can normalize the effect size by some measure of the effect variance, to get a t-statistic. That's generally what's done in most neuroimaging programs these days.

The big reason to extract beta weights at all is that they give you a numerical estimate of the effect size of a particular condition. If the beta for A at a given point for one subject is three times that at the same point for another subject, you know that A had three times a bigger effect in the first subject. This can be an ideal measure to use in regressions against some behavioral measure. For example, you might want to know if a given's subject's self-reported difficulty with a task correlated with the size of the effect of that condition in a particular region. You can extract the beta weights from that regions for each subject, run a simple regression, and find how significant it comes out.

You can also use beta weights to correlate with each other as a crude way of indexing connections between regions. If subjects with anterior cingulates that responded more to condition A also had cerebellums that responded less to condition A, and that correlation is significant across your subject pool, that may tell you something about how the cingulate and cerebellum relate in your task. Check out `Connectivity`_ for more info that direction.

Essentially, beta weights can be used in a myriad of ways - any time you'd like to have some numerical estimate of a given effect or contrast size, rather than simply a statistical measure of the activation.

7. How do I find beta weights / etc.?
*********************
Depends on the program, but every major neuroimaging program can create, in the process of running a GLM, an image of the voxel-by-voxel beta weights for each condition. In SPM, these are the beta_00*.img files produced by estimating a model; in AFNI, they're the fit-coefficient or parameter images that can be produced as part of the bucket dataset output. Other programs generally have similar names for these images. Getting the beta weights is simply a matter of using some image extraction utility - something like roi_extract for SPM - to get the voxel-by-voxel intensity values in your desired ROI. These can then be averaged across the ROI, or you can look only at the peak beta value, etc. Check out `ROI`_ for more.

8. How do I combine information from an ROI across the whole thing?
*********************
The most common strategy in dealing with multiple voxels in an ROI is simply to average your measure across all voxels in the ROI. This has the advantage of being simple to do and simple to explain in a paper. Friston et al. (2006) point out this method may be too conservative; if the ROI has any heterogeneity (say, half of it activates and half of it de-activates), you'll tend to miss things. More complicated methods can be used to identify different subsets of voxels within the ROI with separable responses. Taking the first eigenvariate of the response across voxels from a principal components analysis of the ROI is a simple version of this (and supported in SPM).


Random and Fixed Effects
===================

1. What is a random-effects analysis? What's a fixed-effects analysis? What's the difference?
*********************
Random-effects and fixed-effects analyses are common concepts in social science statistics, so there are a lot of good intros to them out on the web. But in a very small nutshell: A fixed-effects analysis assumes that the subjects you're drawing measurements from are fixed, and that the differences between them are therefore not of interest. So you can look at the variance within each subject all lumped in together - essentially assuming that your subjects (and their variances) are identical. By contrast, a random-effects analysis assumes that your measurements are some kind of random sample drawn from a larger population, and that therefore the variance between them is interesting and can tell you something about the larger population.

Perhaps the most fundamental difference between them is of inference. A fixed-effects analysis can only support inference about the group of measurements (subjects, etc.) you actually have - the actual subject pool you looked at. A random-effects analysis, by contrast, allows you to infer something about the population from which you drew the sample. If the effect size in each subject relative to the variance between your subjects is large enough, you can guess (given a large enough sample size) that your population exhibits that effect - which is crucial for many group neuroimaging studies.

2. So what does the difference between them mean for neuroimaging data?
*********************
If you're interested in making any inferences about the population at large, you essentially are required to do some kind of random-effects analysis at some point in your stream. Not all studies demand this - some types of patient studies, for example - but in general, a random-effects analysis will take place at some point. However, random-effects analyses tend to be less powerful for neuroimaging studies, because they only have as many degrees of freedom as number of subjects. In most neuroimaging studies, you have vastly more functional images per subject than you do subjects, and so you have vastly more degrees of freedom in a fixed-effects analysis.

3. In what situations are each appropriate for neuroimaging analysis?
*********************
Generally, a neuroimaging study with more than one or two subjects will have a place for both types of analysis. The typical study proceeds with a type of model called the hierarchical model, in which both fixed and random effects are considered, but the two types of factors are limited and entirely separable. Single-subject analyses are generallly carried out with a fixed-effects model, where only the scan-to-scan variance is considered. Those analyses generally yield some type of summary measure of activation, be it a T-statistic or beta weight or other statistic. Once those summary measures are collected for each subject, then, a random-effects analysis can be performed on the summaries, looking at the variance between effect sizes as a random effect. Again, only a single source of variance is considered at a single time.

For the most part, the rule of thumb is: single-subject analyses should be fixed-effects (to leverage the greater power of a fixed-effects model) and any analysis involving a group of subjects that you'd like to express something about the population should be random-effects.

4. How do I carry out a fixed-effects analysis in AFNI/SPM/BrainVoyager?
*********************
Generally, the standard single-subject model in all neuroimaging software is a fixed-effects model. Only a single source of variance is considered - the variance between scans (or points in time). If you include several subjects' functional images in a single fMRI model (as opposed to basic model) in SPM, for example, the program will run fine - you'll just get a fixed-effects model over several subjects at once. Any program that produces summary statistic images from single subjects will generally be a fixed-effects model: the standard GLM analysis in SPM and BrainVoyager, for example, or 3dFIM+ or 3dDeconvolve in AFNI. All of these apply a fixed-effects model of your experiment to look at scan-to-scan variance for a single subject. Other subjects could be included, as mentioned, but the variance between subjects will not generally be considered.

5. How do I carry out a random-effects analysis in AFNI/SPM/BrainVoyager?
*********************
Until a few years ago, this was a trickier question, but the Holmes & Friston paper highlighted the need for random-effects models in group neuroimaging studies, and since then (and before, in some cases), every major neuroimaging program has made the hierarchical model the default for group analysis. The idea is built into every program and quite simple: once you've got summary images of the effect sizes from each of your subjects (from single voxels or ROIs or whatever), you then simply throw those effect size summaries into a 'basic' statistical test to look for effect size across the effect sizes. The simplest is a one-sample t-test, but more complicated models can also be used: regressions, ANOVAs, etc. In SPM and BrainVoyager, the 'basic models' button or menu will take you to these sorts of group tests; in AFNI, 3dttest is a simple group t-test program, or 3dregana will do group regressions.

6. Which files should I include in my random-effects analysis? Contrast images? T-statistic images? F-statistics? Why one and not the others?
*********************
This is an important point, and explained better by Holmes' random-effects model, which should be required reading for anyone doing a random-effects test. In general, you want to include whatever image is a summary of your effect size, and not a measure of the significance of your effect size. Evaluating the significance of a group of significances is a layer beyond the statistics you're interested in - you want your measurements to really reflect how big the effect was at the ROI or voxel, not anything about the rest of the variance across the brain. So in general, for a t-test contrast, the image you want to include is the contrast image - the weighted sum of your beta weights - or a raw beta image. In SPM, that's the con_00*.img files, or the beta files themselves.

As a side note, for tests of more than one constraint at once - such as F-tests - the proper summary image is actually kind of tricky - simply including the ESS (Extra Sum of Squares) image into a standard random-effects test is not the way to go. SPM has a multivariate toolboox that may be of help in handling group F-tests directly, but more usually, the approach is to figure out what constraint in the F-test is driving the effect and use that constraint's contrast image in the group analysis.


Realignment
===================

1. What is realignment / motion correction?
*********************
In a perfect world, subjects would lie perfectly still in the scanner while you experimented on them. But, of course, in a perfect world, I'd be six foot ten with a killer Jump Hook, and my car would have those hubcaps that spin independently of the wheels. Sadly, I only have three of my original Camry hubcaps, and subjects are too darned alive to hold perfectly still. When your subject moves in the scanner, no matter how much, a couple things happen:

* Your voxels don't move with them. So the center of your voxel 10, 10, 10, say, used to be just on the front edge of your subject's brain, but now they've moved a millimeter backwards - and that same voxel now sampling from outside the brain. If you don't correct for that, you're going to get an blurry-looking brain when you average your functional effect over time. Think of the scanner as a camera taking a really long exposure - if your subject moves during the exposure, she'll look blurry.
* Their movement generates tiny inhomogeneities in the magnetic field. You've carefully prepared your magnetic field to be perfectly smooth everywhere in the scanner - so long as the subject is in a certain position. When she moves, she generates tiny "ripples" in the field that can cause transient artifacts. She also subtly changes the magnetic field for the rest of the experiment; if she gradually moves forward, for example, you may see the back of her brain get gradually brighter over the experiment as she changes the field and moves through it. If you don't account for that, it'll look like the back of her brain is getting gradually more and more active.
* Realignment (also called motion correction - they're the same thing) mainly aims to help correct the first of these problems. Motion correction algorithms look across your images and try to "line up" each functional image with the one before it, so your voxels always sample the same location and you don't get blurring. The second problem is a little trickier - see the question below on including movement parameters in your design matrix. This issue also comes up in correcting for physiological movement, so check out `Physiology and fMRI`_ as well.


2. How do the major programs' algorithms work? How do they perform relative to each other?
*********************
SPM, AFNI, BrainVoyager, AIR 3.0, and most other major programs, all essentially use modifications of the same algorithm, which is the minimization of a least-squares cost function. The algorithm attempts to find the rigid-body movement parameters for each image that minimizes the voxel-by-voxel intensity difference from the reference image.

The particular implementation of the algorithm varies widely between programs, though. Ardekani et. al (below) does a detailed performance breakdown of SPM99 vs. AFNI98 vs. AIR 3.0 vs. TRU. AFNI is by far the fastest and also the most accurate at lower SNRs; SPM99, though slower, is the most accurate at higher SNRs. See below for more detail...

3. How much movement is too much movement?
*********************
Tough to give an exact answer, but Ardekani et. al find that SPM and AFNI can handle up to 10mm initial misalignment (summed across x/y/z dimensions) without significant trouble. Movement in any single dimension greater than the size of a single voxel between to images is probably worth looking at, and several images with greater than one-voxel motion in one run is a good guideline for concern.

4. How should you correct motion outliers? When should you just throw them out?
*********************
Attempting to correct extreme outliers is a tricky process. On the one hand, images with extremely distorted intensities due to motion can have a measurable distortion effect on your results. On the other hand, distinguishing intensity changes due to motion as opposed to task-related signal is by no means an easy process, and removing task-related signal can also measurably worsen your results (relative to both Type I and II errors).

Our current thinking in the lab is that outlier correction should be attempted only when you can find isolated scans who show significantly distorted global intensities (several standard deviations away from the mean) that are with a TR or two of a significant head movement. A significant head movement without a global intensity change is probably handled best by the realignment process itself; a significant intensity change without head motion may have nothing to do with motion.

Another option is to simply censor (i.e., not use) the images identified as iffy; this is easier in AFNI than in SPM. This has the disadvantage of possibly distorting your trial balancing in a given session if whole trials are removed, as well losing whatever task signal there may be in that scan. It has the advantage of being more statistically valid - outlier correction with interpolation obviously introduces new temporal correlation into your timeseries.

Several things might make a particular session entirely unusable: several isolated scans with head motion of greater than 10mm (summed across x/y/z); several scans with head motion in a single direction greater than the size of a single voxels; a run of several scans in a row with significant motion and significant intensity change; high correlation of your motion parameters with your task (see below). All subjects should be vetted for these problems before their results are taken seriously...

5. How can you tell if it's working? Not working?
*********************
Realignment in general is pretty robust; the least-squares algorithm will always produce some solution. It may, however, get caught in a non-optimal solution, particularly with scans that have a lot of motion and/or a big intensity change from the one before. It's difficult to evaluate realignment's effects post hoc just by looking at your results; the best way to make sure it's worked is visual inspection. SPM's "Check Reg" button will allow you to simultaneously display up to 15 images at once, side-by-side with a crosshair placed identically in each of them, to make sure a given voxel location lines up among several images. You may want to look particularly at scans you've identified with significant head motion, as well as comparing the first and last images in your run...

6. Should I include movement parameters in my design matrix? Why or why not?
*********************
In a nutshell, even after realignment, various effects like interpolation errors, changes in the shim from head movement, spin history effects, etc. can induce motion-correlated intensity changes in your data. Including your motion parameters in your design matrix can do a very good job of removing these changes; including values derived from these parameters, such as their temporal derivative, or sines of their values (see Grootonk et. al) can do an even better job.

The reason not to include these parameters is that there's a pretty good chance you also have actual task-related signal that happens to correlate with your motion parameters, and so removing all intensity changes correlated with motion may also significantly decrease your sensitivity to task-related signal. Before including these parameters in your matrix, you're probably wise to check how much your motion correlates with your task to make sure you're not inadvertantly removing the signal you want to detect.

7. What is 'unwarping'? Why is it included as a realignment option in SPM2? And when should I use it?
*********************
Head motion can cause artifacts for a variety of reasons - gradual changes in the shim, changes in dropout, changes in slice interpolations, spin-history effects - but certainly one of the big ones is motion-by-susceptibility interactions. In areas typically vulnerable to susceptibility-induced artifacts - artifacts caused by magnetic field distortion due to air-tissue interfaces - head motion can cause majors changes in those susceptibility regions, and intensities around the interface can change in unpredictable ways as the head moves. The guys at SPM describe it as being like a funhouse mirror near the interface - there's usually some spot of blackout (signal dropout) where susceptibility is really bad, but right around it, the image is distorted in weird ways, and sliding around can change those distortions in unpredictable ways.

Motion-by-susceptbility interaction is certainly one of the biggest sources of motion-related artifact, and some people think it's THE biggest, and so the "unwarp" utility in SPM2 is an attempt to specifically address it. Even if you get the head lined up image-to-image (realignment), these effects will remain, and so you can try and remove them additionally (unwarping). This is essentially a slightly less powerful but very much more specific version of including your motion parameters in the design matrix - you'll avoid almost all the problems about task-correlated motion that go with including your motion parameters as correlates, but (hopefully) you'll get almost all the same good effects. The benefits will be particularly noticeable in high-susceptibility regions (amygdala, OFC, MTL).

One BIG caveat about unwarping, though - as it's currently implemented, I believe it's ONLY usable for EPI images, NOT for spiral. So if you use spiral images, you shouldn't use this. But if you use EPI, it can be worth a try, particularly if you're looking at high-susceptibility regions.

8. What's the best base image to realign to? Is there any difference between realigning to an image at the beginning of the run and one in the middle of the run?
*********************
Not a huge difference, if any. AFNI has a program (findmindiff, I think) that identifies the image in a particular series that has the least difference from all the other images, which would be the ideal one to use. In practice, though, there's probably no significant difference between using that and simply realigning to the first image of the run, unless you have very large (10mm+) movement over the course of the run, in which case the session is probably of questionable use as well...

9. When is realigning a bad idea?
*********************
The trouble with the least-squares algorithm that realignment programs use is that it's easily fooled into thinking differences in intensity between images are all due to motion. If those differences are due to something else - task-related signal, or sudden global intensity changes - the realignment procedure can be fooled and come up with a bad realignment. If the realignment is particularly bad, it can completely obscure your signal, or (arguably worse) generate false activations! This is most pressing in the case of task-correlated motion (see below for discussion), but if you have significant global intensity shifts during your session that aren't motion-related, your realignment will probably introduce - rather than remove - error into your experiment. There are other realignment methods you can use to get around this, but they're slow. See Friere & Mangin, and the `Coregistration`_ page.

10. What can I do about task-correlated motion? What's the problem with it?
*********************
See Bullmore et. al, Field et. al, and Friere & Mangin for more details about this issue. The basic problem stems from the fact that head motion doesn't just rotate and shift the head in an intensity-invariant fashion. Head motion actually changes the image intensities, due to inhomogeneity in the magnetic field, changes in susceptibility, spin history effects, etc. If your subject's head motions are highly correlated with your task onsets or offsets, it can be impossible to how much of a given intensity change is due to head motion and how much is due to actual brain activation. The effect is that task-correlated motion can induce signficant false activations in your data. Including your motion parameters in your design matrix in this case, to try and account for these intensity changes, will hurt you the other way - you'll end up removing task-correlated signal as well as motion and miss real activations.

The extent of the problem can be significant. Field et. al, using a physical phantom (which doesn't have brain activations) were able to generate realistic-looking clusters of "activation," sometimes of 100+ voxels, with head movements of less than 1mm, simply by making the phantom movements increasingly correlated with their task design. Bullmore et. al point out that patient groups frequently have significant differences in how much task-correlated motion they have relative to normals, which can significantly bias between-group comparisons of activation.

Even worse, the least-squares algorithm commonly used to realign function images is biased by brain activations, because it assumes intensity changes are due to motion and attempts to correct for them. As Friere & Mangin point out, even if there's no motion at all, the presence of significant activations could cause the realignment to report motion parameters that have task-correlated motion!

So what can you do? First and foremost, you should always evaluate how correlated your subjects' motion is with your task - the parameters themselves and linear combinations of them. (An F-test can do this - we'll have a script available for this in the lab shortly.) The correlation of your parameters with your task is hugely more important than the size of your motion in generating false activations. Field demonstrated false activation clusters with correlations above r = 0.52. If your subject exhibits very high correlation, there's not much you can do - they're probably not usable, or at least their results should be taken with a grain of salt. There are some techniques (see Birn et. al, below) that may help distinguish activations from real motion, but they're not perfect...

Bullmore et. al, below, report some ways to account for task-correlated motion that may be useful.

Even without any task-correlated motion, though, you should be aware your motion parameters may be biased, as above, towards reporting a correlation. This is not usually a problem with relatively small activations; it may be bigger with very large signal changes. You can avoid the problem entirely by using a different realignment algorithm - based on mutual information, like the algorithms here (`Coregistration`_) - but these are impossibly slow, and not practically usable.

Among the usable algorithms, Morgan et. al reported SPM99 was the best at avoiding false-positive voxels... Keep an eye out for more robust algorithms coming in the future, though... And you may want to try and use one of the prospective motion correction algorithms, as described in Ward et. al. at.


Scanning
===================

This section is intended to address design-related questions that focus primarily on technical aspects about the scanner - things like TR, pulse sequence, slice thickness, etc. Obviously, setting your scanner parameters is mixed in heavily with your experimental design, so be sure to check out some other design-related pages:

* `Design`_
* `Jitter`_
* `Physiology and fMRI`_

1. What pulse sequence shoudl I use (EPI or spiral)?: What are pros and cons of each? What do each of them get you?
*********************

* EPI: More widely used, and hence supported by all fMRI analysis programs. Some programs (FSL, or SPM's unwarping module, for example) do not support spiral data. Can be subject to less drop-out in some regions than spiral-in or spiral-out data alone. Can be easier to figure out what the slice ordering is.
* Spiral: Properly weighted and combined, spiral in-out shows significantly less signal drop-out and shows significantly greater activations in many areas of the brain, including ventromedial PFC, medial temporal lobe, etc. Effect is even more pronounced at higher field strengths (see Preston et. al). However, it is less widely supported, and Gary's trademark spiral i-o sequence may not even be physically possible on some other institutions' equipment.

2. What should your TR be? What are the tradeoffs, and what's the best tradeoff of coverage vs. speed for different types of analysis?
*********************
Bottom line: TR should be as short as possible, given how many slices you want to cover and the limits of your task. Gary's handout and monograph speak best to this issue, and are good quick reads. Decreasing your TR decreases your signal-to-noise ratio (SNR) in any one functional image, but because you have more images to work with, your overall SNR increases with decreasing SNR. Your TR, however, is limited by how many slices you want to take. On the 3T scanner here at Stanford, using spiral in-out, each slice takes approximately 65 msec/slice (TE = 30 msec), so you can get 15 full slices in 1 second (on the 1.5T, slices take about 75 msec, so you can get 13 full slices in 1 sec.). Your tradeoff is that with fewer slices, you have to either accept less coverage of the brain, or thicker slices, which will have poorer resolution in the z direction.

For certain experiments, then - ones focused on primary sensory on motor cortices, when you don't care about the rest of the brain - you can buy yourself shorter TRs by decreasing your number of slices, or you can increase your number of slices and make them smaller, while keeping your TR constant. Assuming you need full coverage of the brain, you can only decrease your TR by making your slices thicker, which you should do as much as possible within the constraints of your desired resolution.

Alternatively, in experimental designs where you're not particularly focused on timecourse information and where you already have good statistical power - namely, block-design experiments - you may want to get better resolution by increasing your number of slices and hence your TR. In event-related experiments, having a short TR becomes even more imporant, due to the relative lack of experimental power in such designs and relative importance of timecourse information.

3. What should your slice thickness be?
*********************
As thick as you think you can get away with. Increasing your slice thickness allows you to decrease your TR and maintain the same coverage, which is desirable as you get better SNR with decreasing TR. Alternatively, if you need good resolution in all dimensions, you can shrink your slice thickness at the expense of either brain coverage or having a longer TR.

4. What should your slice resolution / voxel size be?
*********************
64 x 64 is standard around here for full-brain coverage. With experiments focusing on smaller areas - primary motor and/or sensory cortices - something else (like 128 x 128) may be useful to get better resolution in a smaller area.

This differs from what size you interpolate your voxels to in normalization, which is covered in `Normalization`_...

5. Should you acquire axially/coronally/something else? How come?
*********************
Big issue here, as I understand it, is that your slices are often thicker than your in-slice voxels, and hence your resolution is often poorest in the direction perpendicular to your slices. (Hence, if you acquire axially, your inferior-superior or z-direction resolution may not be great.) If you have a particular structure of interest, depending on its orientation, you may want to arrange your slices so as to get good resolution in the direction necessary to nail down that structure. Anyone else have any comments on this one?

6. BOLD vs. perfusion: what are pros and cons of each? What sorts of experiments would you use perfusion for?
*********************
Perfusion imaging - in which arterial blood is magnetically 'labeled' with an RF pulse, and then tracked as it moves through the brain - has two main advantages we discussed, one of which is thoroughly discussed in the Aguirre article. That advantage is the relatively different noise profile present within the perfusion signal. Unlike BOLD, perfusion noise doesn't have very much autocorrelation, which isn't by itself anything special, but means that perfusion contains much, much less noise relative to BOLD at very low experimental frequencies. There is more noise in general, though, in perfusion imaging, so in general SNR ratios are better for BOLD. But for experiments with very low task-switching frequency - say, blocks of 60s or more, even up to many minutes or hours - BOLD is almost useless, due to the preponderance of low-frequency noise, whereas the perfusion signal is unchanged. This means that with block lengths of longer than a minute, perfusion imaging is probably a better way to go, and experiment which previously weren't possible - block lengths of several minutes, or task switching taking place over several days - might be designed with perfusion.

Another feature of the noise in perfusion imaging is that it appears to be more reliable across subjects. While SNR within a given subject is higher for BOLD, group SNR appears (with limited data in Aguirre) to be higher in perfusion imaging. More research is needed on this subject, but this relative SNR advantage may be useful for experiments with small numbers of subjects, as across-subject variability is always the largest noise source for BOLD experiments, often by a huge factor.

The other primary advantage of perfusion relative to BOLD imaging is that the perfusion signal is an absolute number, rather than a contrast. Each voxel is given a physiologically intelligible value - amount of cerebral blood flow - which means that it can be especially useful for comparing groups of populations. Comparing the results of a particular contrast in depressed vs. normal subjects might not yield any results, for example, but overall blood flow might just be lower in the absolute in a particular regions for depressed subjects relative to normal subjects - which would be very interesting. The ability to compare baselines in perfusion is a strong case for using it in particular types of experiments where baseline information may be interesting.


Segmentation
===================

1. What is segmentation?
*********************
Segmentation is the process by which you separate your brain pictures into different tissue types. You give the segmentation program a brain image, and it classifies every voxel by tissue type - grey matter, white matter, CSF, skull, etc. Some segmentation algorithms operate on a probabilistic basis rather than a "hard" classification (so one voxel might by 60% likely to be grey, 10% likely to be white, etc.). Some segmentation algorithms go further than tissue type, and classify individual anatomical regions as well. Segmentation algorithms often give back output images, consisting of all the grey voxels in the brain, for example.

A subset of segmentation algorithms focus only on the problem of separating brain from skull tissue; these are often called "skull-stripping" or "brain-extraction" algorithms. The problem of classifying brain from skull is slightly easier than classifying different brain tissue types, but many of the same problems are faced, so we lump them in together with general segmentation algorithms.

2. Why should you segment?
*********************
Lotta reasons. Might be you're interested in the details of the segmentation - how much gray matter is in a particular region, how much white matter, etc. A lot of those analyses fall under the label of voxel-based morphometry (VBM), discussed below in the Ashburner & Friston paper. Alternatively, you might be interested in masking your analysis with one of the segments and only examining activated voxels that are in gray matter in a particular region. You might want to segment only to increase the accuracy of another preprocessing step - you might care that your normalization, say, is especially good in gray matter while you don't care as much about its accuracy in white matter. Simply extracting the brain has even more utility; some analysis programs or preprocessing steps require you to strip skull tissue off the brain before using them. You might simply want to create an analysis mask of all the brain voxels and ignore the other ones.

All of those issues would require you to identify which voxels of your image (almost always anatomical) are gray matter, which are white, and which are CSF or skull or other stuff (or at least which are brain and which are not). You can do this by hand, but it's an arduous and hugely time-consuming process, infeasible for large groups of subjects. Several automated methods are available, though, to do it. Generally, the algorithms take some input image and produce labels for every voxel, assigning them to one of the categories above, or sometimes anatomical labels as well (see below). Alternativey, some algorithms exist that do a "soft" classification and assign each voxel a certain probability of being a particular tissue type. Which you use will depend on exactly what your goals for segmentation are.

Finally, segmentation algorithms are increasingly being used not only to separate tissue types but to automate the production of individualized anatomical ROIs. Would you like to hand-draw your caudate or thalamus, say, but figure it'll take too long or be too hard? Automated segmentation algorithms could be used to simplify the process.

3. What are the problems I might face with segmentation?
*********************
Segmentation algorithms face two big issues: intensity overlap and partial voluming. Intensity overlap refers to the fact that the intensity distributions for different tissue types aren't completely separate - they have significant overlap, such that a bright voxel might be a particularly bright gray matter voxel or, just as easily, a particularly bright white matter voxel. Because all segmentation algorithms have to work with is the intensity value at each voxel (and those around it), this poses a problem for hard classifications. As well, inhomogeneities in the magnetic field, susceptibility-induced magnetic changes or head motion during acquisition can all produce gradual shadings of light or dark in images that can make the different tissue types even harder to distinguish - a particular brightness level might be gray matter at the front of the head, but white matter at the back of the head. One way to address these problems is to take spatial location into account; at the simplest level, voxels can always be assigned a high probability of being the same tissue type as the voxels around them (spatial coherence), or one can use a more sophisticated method like incorporating a full prior probability atlas (see Fischl et. al and Marroquin et. al, below).

Partial voluming refers to the fact that even high-res MRI has a limited spatial resolution, and a given voxel might include signal from several different tissue types to varying degree. This is particularly important along tissue-type borders, where if an algorithm is biased towards one tissue type or another, estimates of one tissue type's volume within an area can be significantly inflated or deflated from reality. One way to address this problem is with "soft" classification - instead of semi-arbitarily assigning voxels to definite tissue types, one can assign voxels probabilities of being in a tissue type, and take that confidence level into account when deciding tissue volume, etc.

4. How are coregistration and segmentation related?
*********************
Fischl et. al make the point that the two processes operate on different sides of the same coin - each one can solve the other. With a perfect coregistration algorithm, you could be maximally confident that you could line up a huge number of brains and create a perfect probability atlas - allowing you the best possible prior probabilities with which to do your segmentation. In order to do a good segmentation, then, you need a good coregistration. But if you had a perfect segmentation, you could vastly improve your coregistration algorithm, because you could coregister each tissue type separately and greatly improve the sharpness of the edges of your image, which increases mutual information.

Fortunately, MI thus far appears to do a pretty good job with coregistration even in unsegmented images, breaking us out of a chicken-and-egg loop. But future research on each of these processes will probably include, to a greater and greater extent, the other process as well.

Check out `Coregistration`_ for more info on coregistration...


Slice Timing
===================

1. What is slice timing correction? What's the point?
*********************
In multi-shot EPI (or spiral methods, which mimic them on this front), slices of your functional images are acquired throughout your TR. You therefore sample the BOLD signal at different layers of the brain at different time points. But you'd really like to have the signal for the whole brain from the same time point. If a given region that spans two slices, for example, all activates at once, you want to see what the signal looks like from the whole region at once; without correcting for slice timing, you might think the part of the region that was sampled later was more active than the part sampled earlier, when in fact you just sampled from the later one at a point closer to the peak of its HRF.

What slice-timing correction does is, for each voxel, examine the timecourse and shift it by a small amount, interpolating between the points you ACTUALLY sampled to give you back the timecourse you WOULD have gotten had you sampled every voxel at exactly the same time. That way you can make the assumption, in your modeling, that every point in a given functional image is the actual signal from the same point in time.

2. How does it work?
*********************
The standard algorithm for slice timing correction uses sinc interpolation between time points, which is accomplished by a Fourier transform of the signal at each voxel. The Fourier transform renders any signal as the sum of some collection of scaled and phase-shifted sine waves; once you have the signal in that form, you can simply shift all the sines on a given slice of the brain forward or backward by the appropriate amount to get the appropriate interpolation. There are a couple pitfalls to this technique, mainly around the beginning and end of your run, highlighted in Calhoun et. al below, but these have largely been accounted for in the currently available modules for slice timing correction in the major programs.

3. Are there different methods or alternatives and how well do they work?
*********************
One alternative to doing slice-timing correction, detailed below in Henson et. al, is simply to model your data with an HRF that accounts for significant variability in when your HRFs onset - i.e., including regressors in your model that convolve your data with both a canonical HRF and with its first temporal derivative, which is accomplished with the 'hrf + temporal derivative' option in SPM. In terms of detecting sheer activation, this seems to be effective, despite the loss of some degrees of freedom in your model; however, your efficiency in estimating your HRF is very significantly reduced by this method, so if you're interested in early vs. late comparisons or timecourse data, this method isn't particularly useful.

Another option might be to include slice-specific regressors in your model, but I don't know of any program that currently implements this option, or any papers than report on it...

4. When should you use it?
*********************
Slice timing correction is primarily important in event-related experiments, and especially if you're interested in doing any kind of timecourse analysis, or any type of 'early-onset vs. late-onset' comparison. In event-related experiments, however, it's very important; Henson et. al show that aligning your model's timescale to the top or bottom slice can results in completely missing large clusters on the slice opposite to the reference slice without doing slice timing correction. This problem is magnified if you're doing interleaved EPI; any sequence that places adjacent slices at distant temporal points will be especially affected by this issue. Any event-related experiment should probably use it.

5. When is it a bad idea?
*********************
It's never that bad an idea, but because the most your signal could be distorted is by one TR, this type of correction isn't as important in block designs. Blocks last for many TRs and figuring out what's happening at any given single TR is not generally a priority, and although the interpolation errors introduced by slice timing correction are generally small, if they're not needed, there's not necessarily a point to introducing them. But if you're interested in doing any sort of timecourse analysis (or if you're using interleaved EPI), it's probably worthwhile.

6. How do you know if it’s working?
*********************
Henson et. al and both Van de Moortele papers below have images of slice-time-corrected vs. un-slice-time-corrected data, and they demonstrate signatures you might look for in your data. Main characteristics might be the absence of significant differences between adjacent slices. I hope to post some pictures here in the next couple weeks of the SPM sample data, analyzed with and without slice timing correction, to explore in a more obvious way.

7. At what point in the processing stream should you use it?
*********************
This is the great open question about slice timing, and it's not super-answerable. Both SPM and AFNI recommend you do it before doing realignment/motion correction, but it's not entirely clear why. The issue is this:

* If you do slice timing correction before realignment, you might look down your non-realigned timecourse for a given voxel on the border of gray matter and CSF, say, and see one TR where the head moved and the voxel sampled from CSF instead of gray. This would results in an interpolation error for that voxel, as it would attempt to interpolate part of that big giant signal into the previous voxel.
* On the other hand, if you do realignment before slice timing correction, you might shift a voxel or a set of voxels onto a different slice, and then you'd apply the wrong amount of slice timing correction to them when you corrected - you'd be shifting the signal as if it had come from slice 20, say, when it actually came from slice 19, and shouldn't be shifted as much. There's no way to avoid all the error (short of doing a four-dimensional realignment process combining spatial and temporal correction - possibly coming soon), but I believe the current thinking is that doing slice timing first minimizes your possible error. The set of voxels subject to such an interpolation error is small, and the interpolation into another TR will also be small and will only affect a few TRs in the timecourse. By contrast, if one realigns first, many voxels in a slice could be affected at once, and their whole timecourses will be affected. I think that's why it makes sense to do slice timing first. That said, here's some articles from the SPM e-mail list that comment helpfully on this subject both ways, and there are even more if you do a search for "slice timing AND before" in the archives of the list.

8. How should you choose your reference slice?
*********************
You can choose to temporally align your slices to any slice you've taken, but keep in mind that the further away from the reference slice a given slice is, the more it's being interpolated. Any interpolation generates some small error, so the further away the slice, the more error there will be. For this reason, many people recommend using the middle slice of the brain as a reference, minimizing the possible distance away from the reference for any slice in the brain. If you have a structure you're interested in a priori, though - hippocampus, say - it may be wise to choose a slice close to that structure, to minimize what small interpolation errors may crop up.

9. Is there some systematic bias for slices far away from your reference slice, because they're always being sampled at a different point in their HRF than your reference slice is?
*********************
That's basically the issue of interpolation error - the further away from your reference slice you are, the more error you're going to have in your interpolation - because your look at the "right" timepoint is a lot blurrier. If you never sample the slice at the top of the head at the peak of the HRF, the interpolation can't be perfect there if you're interpolating to a time when the HRF should be peaking - but hopefully you have enough information about your HRF in your experiment to get a good estimation from other voxels. It's another argument for choosing the middle slice in your image - you want to get as much brain as possible in an area of low interpolation error (close to the reference slice).

10. How can you be sure you're not introducing more noise with interpolation errors than you're taking out with the correction?
*********************
Pretty good question. I don't know enough about signal processing and interpolation to say exactly how big the interpolation errors are, but the empirical studies below seem to show a significant benefit in detection by doing correction without adding much noise or many false positive voxels. Anyone have any other comments about this?


Smoothing
===================

1. What is smoothing?
*********************
"Smoothing" is generally used to describe spatial smoothing in neuroimaging, and that's a nice euphamism for "blurring." Spatial smoothing consists of applying a small blurring kernel across your image, to average part of the intensities from neighboring voxels together. The effect is to blur the image somewhat and make it smoother - softening the hard edges, lowering the overall spatial frequency, and hopefully improving your signal-to-noise ratio.

2. What's the point of smoothing?
*********************
Improving your signal to noise ratio. That's it, in a nutshell. This happens on a couple of levels, both the single-subject and the group.

At the single-subject level: fMRI data has a lot of noise in it, but studies have shown that most of the spatial noise is (mostly) Gaussian - it's essentially random, essentially independent from voxel to voxel, and roughly centered around zero. If that's true, then if we average our intensity across several voxels, our noise will tend to average to zero, whereas our signal (which is some non-zero number) will tend to average to something non-zero, and presto! We've decreased our noise while not decreasing our signal, and our SNR is better. (Desmond & Glover demonstrate this effect with real data.)

At the group level: Anatomy is highly variable between individuals, and so is exact functional placement within that anatomy. Even with normalized data, there'll be some good chunk of variability between subjects as to where a given functional cluster might be. Smoothing will blur those clusters and thus maximize the overlap between subjects for a given cluster, which increases our odds of detecting that functional cluster at the group level and increasing our sensitivity.

Finally, a slight technical note for SPM: Gaussian field theory, by which SPM does p-corrections, is based on how smooth your data is - the more spatial correlation in the data, the better your corrected p-values will look, because there's fewer degree of freedom in the data. So in SPM, smoothing will give you a direct bump in p-values - but this is not a "real" increase in sensitivity as such.

3. When should you smooth? When should you not?
*********************

**Smoothing is a good idea if:**

* You're not particularly concerned with voxel-by-voxel resolution.
* You're not particularly concerned with finding small (less than a handful of voxels) clusters.
* You want (or need) to improve your signal-to-noise ratio.
* You're averaging results over a group, in a brain region where functional anatomy and organization isn't precisely known.
* You're using SPM, and you want to use p-values corrected with Gaussian field theory (as opposed to FDR).

**Smoothing'd not a good idea if:**

* You need voxel-by-voxel resolution.
* You believe your activations of interest will only be a few voxels large.
* You're confident your task will generate large amounts of signal relative to noise.
* You're working primarily with single-subject results.
* You're mainly interested in getting region-of-interest data from very specific structures that you've drawn with high resolution on single subjects.

4. At what point in your analysis stream should you smooth?
*********************
The first point at which it's obvious to smooth is as the last spatial preprocessing step for your raw images; smoothing before then will only reduce the accuracy of the earlier preprocessing (normalization, realignment, etc.) - those programs that need smooth images do their own smoothing in memory as part of the calculation, and don't save the smoothed versions. One could also avoid smoothing the raw images entirely and instead smooth the beta and/or contrast images. In terms of efficiency, there's not much difference - smoothing even hundreds of raw images is a very fast process. So the question is one of performance - which is better for your sensitivity?

Skudlarski et. al evaluated this for single-subject data and found almost no difference between the two methods. They did find that multifiltering (see below) had greater benefits when the smoothing was done on the raw images as opposed to the statistical maps. Certainly if you want to use p-values corrected with Gaussian field theory (a la SPM), you need to smooth before estimating your results. It's a bit of a toss-up, though...

5. How do you determine the size of your kernel? Based on your resolution? Or structure size?
*********************
A little of both, it seems. The matched filter theorem, from the signal processing field, tells us that if we're trying to recover a signal (like an activation) in noisy data (like fMRI), we can best do it by smoothing our data with a kernel that's about the same size as our activation.

Trouble is, though, most of us don't know how big our activations are going to be before we run our experiment. Even if you have a particular structure of interest (say, the hippocampus), you may not get activation over the whole region - only a part.

Given that ambiguity, Skudlarski et. al introduce a method called multifiltering, in which you calculate results once from smoothed images, and then a second set of results from unsmoothed images. Finally, you average together the beta/con images from both sets of results to create a final set of results. The idea is that the smoothed set of results preferentially highlight larger activations, while the unsmoothed set of results preserve small activations, and the final set has some of the advantages of both. Their evaluations showed multifiltering didn't detect larger activations (clusters with radii of 3-4 voxels or greater) as well as purely smoothed results (as you might predict) but that over several cluster sizes, multifiltering outperformed traditional smoothing techniques. Its use in your experiment depends on how important you consider detecting activations of small size (less than 3-voxel radius, or about).

Overall, Skudlarski et. al found that over several cluster sizes, a kernel size of 1-2 voxels (3-6 mm, in their case) was most sensitive in general.

A good rule of thumb is to avoid using a kernel that's significantly larger than any structure you have a particular a priori interest in, and carefully consider what your particular comfort level is with smaller activations. A 2-voxel-radius cluster is around 30 voxels and change (and multifiltering would be more sensitive to that size); a 3-voxel-radius cluster is 110 voxels or so (if I'm doing my math right). 6mm is a good place to start. If you're particularly interested in smaller activations, 2-4mm might be better. If you know you won't care about small activations and really will only look at large clusters, 8-10mm is a good range.

6. Should you use a different kernel for different parts of the brain?
*********************
It's an interesting question. Hopfinger et. al find that a 6mm kernel works best for the data they examine in the cortex, but a larger kernel (10mm) works best in subcortical regions. This might be counterintuitive, considering the subcortical structures they examine are small in general than large cortical activations - but they unfortunately don't include information about the size of their activation clusters, so the results are difficult to interpret. You might think a smaller kernel in subcortical regions would be better, due to the smaller size of the structures.

Trouble is, figuring out exactly which parts of the brain to use a different size of kernel on presupposes a lot of information - about activation size, about shape of HRF in one region vs. another - that pretty much doesn't exist for most experimental set-ups or subjects. I would tend to suggest that varying the size of the kernel for different regions is probably more trouble than it's worth at this point, but that may change as more studies come out about HRFs in different regions and individualized effects of smoothing. See Kiebel and Friston, though, for some advanced work on changing the shape of the kernel in different regions...

7. What does it actually do to your activation data?
*********************
About what you'd expect - preferentially brings out larger activations. Check out White et. al for some detailed illustrations. We hope to have some empirical results and maybe some pictures up here in the next few weeks...

8. What does it do to ROI data?
*********************
Great question, and not one I've got a good answer for at the moment. One big part of the answer will depend on the ratio of your smoothing kernel size to your ROI size. Presumably, assuming your kernel size is smaller than your ROI, it may help improve SNR in your ROI, but if the kernel and ROI are similar sizes, smoothing may also blur the signal such that your structure contains less activation. With any luck, we can do a little empirical testing on this questions and have some results up here in the future...


SPM in a Nutshell
===================

SPM is a software package designed to analyze brain imaging data from PET or fMRI and output a variety of statistical and numerical measures that tell you, the researcher, what parts of your subjects' brains were signficantly "activated" by different conditions of your experiment. There are a couple of phases of analyzing data with SPM: spatial preprocessing, model estimation, and results exploration. This page aims to give you a nutshell explanation of what's actually happening in each of those phases (particularly model estimation) and what some of the files floating around your results directories are for. Links to pages with more detail about each aspect of the analysis are down at the bottom of this page.

Spatial preprocessing is conceptually the most straightforward part of SPM analysis. During this phase, you can align your images with each other, warp them (normalize) so that each subject's anatomy is roughly the same shape, correct them for differences in slice time acquisition, and smooth them spatially.

These steps are used for a couple of reasons. Registration and normalization aim to line images from a single subject up (since subjects' heads move slightly during the experiment) and normalization aims to stretch and squeeze the shape of the images so that their anatomy roughly matches a standard template; both of these aim to make localizing your activations easier and more meaningful, by making individual voxels' locations in a given image file match up in a standard way to a particular anatomical location. Slice timing correction and smoothing both enable SPM to make certain assumptions about the data images - that each whole image occurred at a particular point in time (as opposed to slices being taken over the course of an image acquisition, or TR), and that noise in an image is distributed in a relatively random and independent fashion (as opposed to being localized).

Model estimation is the heart of the SPM program, and it's also the most conceptually complex. What you as a researcher want to know from your data is essentially: what (if any) parts of my subject's brain were brighter during one part of my experiment relative to another part? Another way of putting this question might be this: You as a researcher have a hypothesis or model of what happened in an experiment; you have a list of different conditions and when each of them took place, and your model of the person's brain is that there was some kind of reaction in the brain for every stimulus that happened. How good a fit, then, does your hypothetical model provide to the actual MRI data you saw from the person's brain? Specifically, are there particular locations in the brain where your model was a very good fit, and others where it wasn't a good fit? The main work SPM does is to try and find those locations, because locations where your hypothesis proves to be a good fit can be described as "responding" somehow to the conditions in your experiment.

When SPM estimates a model, what it's doing is essentially a huge multiple regression at each voxel of your subject's brain, to see how well the data across the experiment fits your hypothesis, which you describe to SPM as a design matrix. When you tell SPM what your conditions are, what your onset vectors are, etc., it sets up this matrix as a guess at what contribution each condition might make to every image in your experiment. As part of that guess, it automatically convolves the effects of the hemodynamic response function with your stimulus vectors, as well as doing some temporal filtering to make sure it ignores changes in the data that aren't relevant to your conditions.

Once the design matrix has been set up, SPM walks through each voxel in the brain, and does a multiple regression on the data at that point that estimates how much of a contribution every condition in your experiment made to the data and how much error was left over after all the conditions you specified are taken into account. The fit of this regression line is important - how much error is left at the voxel tells you how good your model was - but for most purposes, the actual slope of the regression line is more important. A large positive or negative partial regression slope for a given condition tells us that that condition had a large influence in determining the data at that voxel. This slope, called the beta weight or parameter weight for that condition, is saved by SPM in the beta_* images - one for each column of the design matrix, where each voxel gives the beta weight for that condition at that point.

After the model estimation is complete, you now have a set of data telling you how big an effect each condition of your experiment had at each voxel. By itself, this information may not be useful - in most fMRI experiments, any condition by itself accounts for a tiny portion of the variance. What is useful to know, though, is if one condition made a significantly greater contribution than another condition did. This is where results analysis, and specifically contrast analysis, comes in. When you evaluate your results, SPM asks you to specify a contrast in terms of weights for each conditions. If you have only two conditions in your experiment, assuming your design matrix was (A B), then a contrast vector of (1 -1) would tell SPM you wanted to see at which voxels A had a signficantly larger contribution to brain activity than B did. SPM takes this contrast vector and literally uses it to make a weighted sum of the beta images it's just created; this new image, created by giving each beta image the weight you specified and adding them together, is a con_* image. SPM then looks across the con image at the distribution of weighted parameter values, and combines them with its estimate of the leftover variance from the model estimation, and assigns every voxel a T-statistic (creating an spm_T* image). When you ask SPM to only show you the voxels that are significantly active at a certain p-threshold, it looks at that T-stat image and finds only the voxels whose t-statistics are so large as to fit above that probability threshold - voxels where their weighted parameter values were so large as to be statistically unlikely at your specified level of significance.

Those voxels are where brain activity in your experiment was heavily influenced by one condition in your experiment more than another condition, so heavily influenced as to make it unlikely that activity was just noise. Those voxels were brighter / more intense in one condition of your experiment than they were in another with great reliability, and so they're considered active.

For more detail about a particular aspect of spatial preprocessing, check out the individual FAQ pages at:

* `Coregistration`_
* `Realignment`_
* `Normalization`_
* `Smoothing`_
* `Slice Timing`_


Temporal Filtering
===================

1. Why do filtering? What’s it buy you?
*********************
Filtering in time and/or space is a long-established method in any signal detection process to help "clean up" your signal. The idea is if your signal and noise are present at separable frequencies in the data, you can attenuate the noise frequencies and thus increase your signal to noise ratio.

One obvious way you might do this is by knocking out frequencies you know are too low to correspond to the signal you want - in other words, if you have an idea of how fast your signal might be oscillating, you can knock out noise that is oscillating much slower than that. In fMRI, noise like this can have a number of courses - slow "scanner drifts," where the mean of the data drifts up or down gradually over the course of the session, or physiological influences like changes in basal metabolism, or a number of other sources. This type of filtering is called "high-pass filtering," because we remove the very low frequencies and "pass through" the high frequencies. Doing this in the spatial domain would correspond to highlighting the edges of your image (preserving high-frequency information); in the temporal domain, it corresponds to "straightening out" any large bends or drifts in your timecourse. Removing linear drifts from a timecourse is the simplest possible high-pass filter.

Another obvious way you might do this would be the opposite - knock out the frequencies you know are too high to correspond to your signal. This removes noise that is oscillating much faster than your signal from the data. This type of filtering is called "low-pass filtering," because we remove the very high frequencies and "pass through" the low frequencies. Doing this in the spatial domain is simply spatial smoothing (see `Smoothing`_); in the temporal domain, it corresponds to temporal smoothing. Low-pass filtering is much more controversial than high-pass filtering.
Finally, you could apply combinations of these filters to try and restrict the signal you detect to a specific band of frequencies, preserving only oscillations faster than a certain speed and slower than a certain speed. This is called "band-pass filtering," because we "pass through" a band of frequencies and filter everything else out, and is usually implemented in neuroimaging as simply doing both high-pass and low-pass filtering separately.

In all of these cases, the goal of temporal filtering is the same: to apply our knowledge about what the BOLD signal "should" look like in the temporal domain in order to remove noise that is very unlikely to be part of the signal. This buys us better SNR, and a much better chance of detecting real activations and rejecting false ones.

2. What actually happens to my signal when I filter it? How about the design matrix?
*********************
Filtering is a pretty standard mathematical operation, so all the major neuroimaging programs essentially do it the same way. We'll use high-pass as an example, as low-pass is no longer standard in most neuroimaging programs. At some point before model estimation, the program will ask the user to specify a cutoff parameter in Hz or seconds for the filter. If specified in seconds, this cutoff is taken to mean the period of interest of the experiment; frequencies that repeat over a timescale longer than the specified cutoff parameter are removed. Once the design matrix is constructed but before model estimation has begun, the program will filter each voxel's timecourse (the filter is generally based on some discrete cosine matrix) before submitting it to the model estimation - usually a very quick process. A graphical representation of the timecourse would show a "straightening out" of the signal timecourse - oftentime timecourses will have gradual linear drifts or quadratic drifts, or even higher frequency but still gradual bends, which are all flattened away after the filtering.

Other, older methods for high-pass filtering simply included a set of increasing-frequency cosines in the design matrix (see Holmes et. al below), allowing them to "soak up" low-frequency variance, but this is generally not done explicitly any more.

Low-pass filtering proceeds much the same way, but the design matrix is also usually filtered to smooth out any high frequencies present in it, as the signal to be detected will no longer have them. Low-pass filters are less likely to be specified merely with a lower-bound period-of-interest cutoff; oftentimes low-pass filters are constructed deliberately to have the same shape as a canonical HRF, to help highlight signal with that shape (as per the matched-filter theorem).

3. What’s good about high-pass filtering? Bad?
*********************
High-pass filtering is relatively uncontroversial, and is generally accepted as a good idea for neuroimaging data. One big reason for this is that the noise in fMRI isn't white - it's disproportionately present in the low frequencies. There are several sources for this noise (see `Physiology and fMRI`_ and `Basic Statistical Modeling`_ for discussions of some of them), and they're expressed in the timecourses sometimes as linear or higher-order drifts in the mean of the data, sometimes as slightly faster but still gradual oscillations (or both). What's good about high-pass filtering is that it's a straightforward operation that can attenuate that noise to a great degree. A number of the papers below study the efficacy of preprocessing steps, and generally it's found to significantly enhance one's ability to detect true activations.

The one downside of high-pass filtering is that it can sometimes be tricky to select exactly what one's period of interest is. If you only have a single trial type with some inter-trial interval, then your period of interest of obvious - the time from one trial's beginning to the next - but what if you have three or four? Or more than that? Is it still the time from one trial to the next? Or the time from one trial to the next trial of that same type? Or what? Skudlarski et. al point out that a badly chosen cutoff period can be significantly worse than the simplest possible temporal filtering, which would just be removing any linear drift from the data. If you try and detect an effect whose frequency is lower than your cutoff, the filter will probably knock it completely out, along with the noise. On the other hand, there's enough noise at low frequencies to almost guarantee that you wouldn't be able to detect most very slow anyways. Perfusion imaging does not suffer from this problem, one of its benefits - the noise spectrum for perfusion imaging appears to be quite flat.

4. What’s good about low-pass filtering? Bad?
*********************
Low-pass filtering is much more controversial in MRI, and even in the face of mounting empirical evidence that it wasn't doing much good, the SPM group long offered some substantial and reasonable arguments in favor of it. The two big reasons offered in favor of low-pass filtering broke down as:

* The matched-filter theorem suggests filtering our timecourse with a filter shaped like an HRF should enhance signals of that shape relative to the noise, and
* We need to modify our general linear model to account for all the autocorrelation in fMRI noise; one way of doing that is by conditioning our data with a low-pass filter - essentially 'coloring' the noise spectrum, or introducing our own autocorrelations - and assuming that our introduced autocorrelation 'swamps' the existing autocorrelations, so that they can be ignored. (See `Basic Statistical Modeling`_ for more on this.) This was a way of getting around early ignorance about the shape of the noise spectrum in fMRI and avoiding the computational burden of approximating the autocorrelation function for each model. Even as those burdens began to be overcome, Friston et. al pointed out potential problems with pre-whitening the data as opposed to low-pass filtering, relating to potential biases of the analysis.

However, the mounting evidence demonstrating the failure of low-pass filtering, as well as advances in computation speed enabling better ways of dealing with autocorrelation, seem to have won the day. In practice, low-pass filtering seems to have the effect of greatly reducing one's sensitivity to detecting true activations without significantly enhancing the ability to reject false ones (see Skudlarksi et. al, Della-Maggiore et. al). The problem with low-pass filtering seems to be that because noise is not independent from timepoint to timepoint in fMRI, 'smoothing' the timecourse doesn't suppress the noise but can, in fact, enhance it relative to the signal - it amplifies the worst of the noise and smooths the peaks of the signal out. Simulations with white noise show significant benefits from low-pass filtering, but with real, correlated fMRI noise, the filtering because counter-effective. Due to these results and a better sense now of how to correctly pre-whiten the timeseries noise, low-pass filtering is now no longer available in SPM2, nor is it allowed by standard methods in AFNI or BrainVoyager.

5. How do you set your cutoff parameter?
*********************
Weeeeelll... this is one of those many messy little questions in fMRI that has been kind of arbitrarily hand-waved away, because there's not a good, simple answer for it. You'd to like to filter out as much noise as possible - particularly in the nasty part of the noise power spectrum where the noise power increases abruptly - without removing any important signal at all. But this can be a little trickier than it sounds. Roughly, a good rule of thumb might be to take the 'fundamental frequency' of your experiment - the time between one trial start and the next - and double or triple it, to make sure you don't filter out anything closer to your fundamental frequency.

SPM99 (and earlier) had a formula built in that would try and calculate this number. But if you had a lot of trial types, and some types weren't repeated for very long periods of time, you'd often get filter sizes that were way too long (letting in too much noise). So in SPM2 they scrapped the formula and now issue a default filter size of 128 seconds for everybody, which isn't really any better of a solution.

In general, default sizes of 100 or 128 seconds are pretty standard for most trial lengths (say, 8-45 seconds). If you have particularly short trials (less than 10 seconds) you could probably go shorter, maybe more like 60 or 48 seconds. But this is a pretty arbitrary part of the process. The upside is that it's hard to criticize an exact number that's in the right ballpark, so you probably won't get a paper rejected purely because your filter size was all wrong.
