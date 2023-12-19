# A2

# Inferring Depth from Stereo

### Objectives:
- Given two images, we can find the depth of the image using Naive Stereo and Markov Random Field algorithm. 
- Produces 3D Stereoscopic Image

### Disparity Costs:
Since we are assuming that the images have been already rectified, we can match pixels from the left image to the right image's pixels which is located within the scanline in the same row as left pixels that we are looking. Ideally, we check the whole pixels to attain which pixel that has the best match. However, it is computationally expensive esepecially if we have high resolution images. Therefore, the span of area that the model will search is limitized by MAX_DISPARITY. Let's say if we set MAX_DISPARITY is equal to 10, it means that we search the area start from the same coordiante as the left pixel, scan 9 pixels to the left, and other 9 pixels to the right. Every slide, the disparity costs is calculated and stored in a matrix with a shape of (H,W,D) where H is the height of image, W is the width, and D is the number of location within the scanline in the right image that we checked. In this case, D is equal to MAX_DISPARITY * 2 - 1.

Before the learning process, we normalized the image by dividing each pixels with 255 such that every pixel value has range [0,1]. To reduce the noise, we use a window that will be convolved along the image. Larger window will give smoother result but is unable to capture the local information well. After some trials, we found that window of size 5x5 gives better result. The disparity costs is calculated based on the sum of squared difference between window in the left image and window in the right image. Based on the experiment, truncated quadratic function helps to create better output. We use the alpha value of 0.53 to control the maximum value of qudratic function.

### Naive Stereo:
This algorithm only utilize the disparity costs to find the index of 'D' where the minimum cost is located for every pixels. This index later will be converted into [0, 255] value as an image. To get the index, we have to make a slight adjustment. The principle of depth inference is to figure out the depth based on the shift distance of an object. If the object is close to the camera, it will have greater shift compared to the object that is located far from the camera. We utilize this concept for indexing when the disparity costs is stored in the matrix. The furthest shift should be located at the beginning (index 0) for left direction and at the end of matrix (index MAX_DISPARITY * 2 - 1) for the right direction. The origin (0 offset) is located at the index of MAX_DISPARITY - 1. After finding the index of minimum costs, we substract that index with the origin index (MAX_DISPARITY - 1) and take the absolute value.

### MRF with Loopy Belief Propagation
In MRF, there are some additional terms in the cost function. The transition cost and message cost from neighbors are introduced.
Transition cost is the difference of disparity index whereas the message cost is calculated based on what the neighbors think
of an i th pixel label should be from the previous iteration. Both transition and message cost is normalized for the next
iteration to avoid the overflow and underflow problems. Similar to disparity cost, transition cost is calculated using
truncated quadratic function. As we consider the message from the previous iteration, at the first iteration we initialize
the message with 0. Afterwards, this message will be updated in every iteration. In the message passing process, in each
iteration (let's say at time = t) every pixel calculate the total cost (disparity cost, transition cost, and message cost
from their neighbors except the node that the message is passing to) and pass it to their neighbor in the right, left, 
up, and down direction. To sum up, here is the algorithm:

    for i in range(max_iteration):
        Every pixel calculate total cost and pass the message to the right direction
        Calculate total cost and pass the message to the left direction
        Calculate total cost and pass the message to the up direction
        Calculate total cost and pass the message to the down direction
    end

After the messages converge, the disparity label in each pixel is predicted by finding the minimum value of the sum of
disparity cost, message cost from the all 4 neighbors (neighbor on above, below, left, and right). The same concept is
also applied as in Naive Stereo where the index is substracted with (MAX_DISPARITY - 1) and take the absolute value.

### 3D Stereoscopic Image
As an additional task, we also produce the 3D red-cyan image. The idea is quite simple since the two images are already
given to use. We take the red value of the left image to create red image and merge it with the right image where we use
the green and blue value to generate cyan image. To generate how close or far the 3D image looks, one of the image should be added an offset. The illustration is shown in below:

<img width="1080" alt="Screenshot 2023-04-11 at 23 23 47" src="https://media.github.iu.edu/user/20652/files/2fb1c40a-2d35-4d93-b6bb-e1f93f6ef49e">
(picture from https://eeweb.engineering.nyu.edu/~yao/EL6123old/stereo_new.pdf)

The concept is that if we want to produce the 3D image that appears in front of the screen, the right eye should see the object on the left while left eye should see the object on the right. It is shown in the left picture where the angle between eyes is intersected in front of the screen. The glass has red optic on the right and cyan optic on the left. Therefore, the right image should be in red color as the object shift to left compared to the left image which is transformed into cyan image. The offset is adjusted based on how far from the screen that 3D image is wanted to be. Based on the experiment, we use the offset is 25 pixels such that the 3D image can be seen with a normal reading distance from our eyes to the screen. This value coud vary depends on the image data that we have.

### Problem & Decision:
From the given data, the training images do not require rectification as those left and right images are already taken in the same plane. This means that the camera is pointing the same direction and it only slides from the left to the right. Because of this, the object in the right image will be shifted to the left compared to the left image. Therefore, we can absoultely certain to tell the program that it only needs to check on the left direction starting from the same coordinate from the left image that we want to check, with the number of slide of MAX_DISPARITY - 1. We have tried this approach and it gives better and faster result compared to searching in both direction (left and right). It is better because we force the model to only check to the left and ignore every pixels on the right and it is faster because it requires less iteration. However, it gives problem as this approach is not robust. If the 2 images are taken in different orientation, we need to rectify the image. This will cause local information can be shifted either to the left or to the right. As a final decision, we choose to check both direction for more robust model but with same trade-offs (slower and poorer result).
