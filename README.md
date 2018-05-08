This is my work on the Drop Tests/ Free Spinning tests for the GDP. This repository contatins the data that was collected, and the python code used to process the data.

The data was collected by an Arduino Due with the input pins set up as external interrupts. Three unipolar (either high or low) hall effect sensors were connected straight to the Due, and when their state changed, the Due printed its internal time, in microseconds, to serial. The raw data collected is only a list of the times when a sensor switched state.

#mathing_the_data.py

  This script contains the code used to transform and condition the data from time only, to time and angle of the wheel. In theory, each sensor should have evenly spaced phases, and no switching bias.
  
  In reality, the sensors have an inherent threshold such that the magnetic field needs to be stronger in one direction to switch high, than in the other direction to swing low. In other words, in an even rotating magnetic field, ideally the width of the high pulse and low pulse should be identical, but in practice they are asymetric, due to the bias of the internal transistors and comparators of the sensor.
  
  Also in reality, the sensors will have a slight hysterisis, and more importantly, the position they were mounted in will not be ideal (laser cutting and hot glue do not offer molecular precision). For these reasons, there will be an offset of the phase between each sensor.
  
  mathing_the_data uses optimisation to select the offests that result in the least noisy data. It breaks the raw data in to three streams (every third pulse comes from the same sensor, so this isn't hard), and finds an offset that minimises the error between the angular velocity of the data and the heavily filtered angular velocity of each stream. It then recombines the three sensor streams in a similar way, choosing offests of the second and third streams to minimise the error between the angular velocites of the data and the "filtered" data.
  
  I say "filtered" the second time, because I don't know if it can truly be considered a filter. The filter used in the intra-sensor optimizing is a Savitzky–Golay filter implemented by SciPy, with constants eye-balled to give a smooth outcome. For the inter-sensor optimizing, the "filter" used was actually interpolation. I didn't realise this at the time (yes, I know I went back and explained it in comments), but the reason why it smoothed the data was because it reduced the noise gain of differentiation (F'(z)/F(z) = (z-1)/T, interpolation decreases the sampling period T, reducing overall gain (z-1)/T). So I say "filtered", because I'm not sure that this counts as a filter, although it definately works as one if you apply it before differentiating data.
  
  I admit, my approach to minimizing noise is fairly naive. Optimizing the error between data and filtered data will normalise the data towards the filtered data, which will give incorrect results if the filter is badly applied. I knew this at the time, and tried to mitigate this by re-apply the filter for each offset, but it is still naive. I literally did not know this at the time, and this serves as a testiment to how much I've learned and taught myself over the past few weeks, but a much better approach would have been to use Tikhonov or ridge regularization. I'm still not sure how to select the regularization parameter, but I do know how to get the script to optimise. I'll talk more about regularization in "data prossing.py", since that is where I actually tried to implement it.
    
  Now, there is the obvious question of why go to the effort of optimising the offesets when I'm comfortable using digital filters? The answer is that filtering introduces it's own distortion to the data. They can introduce phasing errors, and they can significantly change the shape of the data by limiting the rate that the data can change. It seems counter-intuitive that filtering is distortion, but filtering involves intentionally changing the shape of the data, and changing the shape of information is the definition of distortion. The heavier the filtering used, the heavy the distortion of the data, and the more likely that the data collected might not be representative of the real world values. By optimizing the offsets, we reduce the noise in the data in a way that respects the sources of noise, and should actually improve how well the collected data represents the real-world physical system being observed. Even if it doesn't improve the accuracy of the data, it shouldn't distort the data, and it does reduce the noise in the data which reduces how heavy any future filtering will need to be. I truly believe that, despite the naivity of the filtering used, I really did manage to reduce the noise in the data without loosing accuracy through filter distortion.

#mathing the data.py

  This is an old version of mathing_the_data.py. I wanted to import this file in to "data processing.py", which I couldn't do with gaps in the name. So I renamed the file to replace the gaps with underscores, and used mathing_the_data.py as the working copy. I ultimately did not import the script into the data processing script in the final version of the "data processing.py", so all I really did was add confusion and stuff to the git repository. So yeah.

#data processing.py

  This is the file where I used the conditioned data to find the constants needed for the friction model. Despite the conditioning applied in mathing_the_data, there still was noise in the data, that was magnified by differentiation. The test models describe acceleration, so either the data must be differentiated twice, or the models must be integrated twice.  The friction model is a non-linear differential equation, and isn't possible to solve analytically, so would have to be solved numerically using Euler's method, which is technically simple to implement, but involves generating huge datasets of over a million values each, every time a variable is changed. Which tends to happen sometimes when you're optimising. Solving for theta would take an inordinate amount of computational resources, which leaves us with the much less intensive differentiation.
  
  The ideal method for differentiation would be to use Cullum's variation on Tikhonov regularization. This method is to optimize every value in the data set u(t), to minimize both the sum of (Au(t) - f(t))^2, and the sum of |u'(t)|, where u(t) = f'(t), and Au(t) is the integral of u(t). Since we are using discrete data, integration can be done by linear algebra. In plain(er) english, we differentiate the data, then change each value to minimize both the eraticism (u'(t)) and how different the integrated differentiated data is from the original data. In actual plain english, we change velocity or acceleration a little bit at a time so that the graph of velocity or acceleration is smooth, but not too different from what it actually is.
  
  Tikhonov regularization is trying to minimize two equations, so instead minimizes the weighted sum of the two equations. How the equations are weighted is still being researched, but it is generally excpeted that, although there are good objective methods to choosing the weighting parameter, trying to get the best weighting is still essentially an artform. I tried to let the script optimize it's own weighting parameter, but it always ended up at the lowest bound I had given it (this was just differentiating, not trying to fit the results to the acceleration models). There was also the issue that, since datasets were quite large (a few thousand datapoints each) performing Tikhonov regularization once was fairly slow (took about 20 minutes each time). Slow code that would have to run overnight was not an option, because by this point in the project there was very little time left, and I couldn't wait half a day to find out I need to tweak something and then wait another half day to find out what else I need to tweak.
  
  The approach used in the final code, was to differentiate the data and apply a first order low-pass filter on the result. I let the code choose it's own cutoff frequencies, because it was differentiating and filtering while also curve fitting to all of the test results simultaneously, i.e. it had to choose cutoffs that minimised the error between the acceleration models and all data.
  
  Cullum's aproach would definately have gotten better results, but it just took way too long to optimise. I'm sure it would have been quicker to use SciPy's Newton-CG method instead of the L-BFGS method it was using, but that requires providing a hessian matrix, which was too complicated for me to do in the short time left in the project.
  
  You'll note that @jit is at the top of most of the functions: this is sets that function to be compiled with a just-in-time compiler provided by numba. The functions can also be cached so that they're not re-compiled every time they're called, and numba can vectorise them sometimes so that the function is executed across all available CPU cores. It can be a pain to get to work sometimes, but for long functions it is probably the single easiest way to improve execution time. I also used NumPy arrays instead of recursive loops anywhere that it was possible to do so, which made the Tikhonov regularization run 20 times faster (though still too slow to be used).
