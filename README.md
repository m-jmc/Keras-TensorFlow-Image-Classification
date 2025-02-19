
***

# **Image Classification: A CUDA/GPU implementation**

***

## Installation 

The initial install of keras and tensorflow via anaconda is somewhat convoluted (pun intended) but be sure to point your library to the correct anaconda environment while following the Nvidia Deep Learning SDK Documentation linked [HERE](https://docs.nvidia.com/deeplearning/sdk/cudnn-install/index.html#install-windows) 
<br>
<br>
As of June 2020 when downloading and installing tensorflow be sure to use version 2.1. The newest version 2.2 does not work with CUDA. The latest cuda version 10.2 does not work with Tensorflow 2.1 and requires CUDA version 10.1.
<br>
<br>
Once that was all sorted, I installed anaconda, created a 'tf' anaconda environment using python 3.6 (the latest python 3.7 doesn't work with... I'm beginning to see a trend at this point.). When attempting to install tensorflow and its dependencies from R I ran into some issues. Rather than continue to wrangle with the R side of it, I found it easier to just -pip install or upgrade/downgrade whatever package was causing the error from the conda envionrment python prompt.
<br>
<br>
What follows has been minimally adapted from numerous other example guides but marks a successful implementation of GPU Deep Learning for future exploration The full analysis took just under 4 hours to run (AMD Ryzen 5 3600, 16GB DDR4 3200, Nvidia RTX 2060 Super 8gb), this upload reflects values obtained during a previous run. 
<br>
<br>

***


```r
# Initial install of packages and specification of tf version shown below, once done, the post install library loads should be all thats needed.

## Initial Install:
# library(reticulate)
# library(keras)
# keras::use_condaenv("tf", required = TRUE)
# install_keras(tensorflow = "2.1.0-gpu")
# library(tensorflow)

# Post install, subsequent runs:
library(reticulate)
library(keras)
keras::use_condaenv("tf", required = TRUE)
library(tensorflow)


# Many of the methods used to check session information no longer work in later versions of tensorflow. Below is what I used to list the session devices. If everything is installed correctly you'll see the CPU and GPU device listed. Keras will use the GPU device by default when it is available. 
# https://stackoverflow.com/questions/38009682/how-to-tell-if-tensorflow-is-using-gpu-acceleration-from-inside-python-shell
sess = tf$compat$v1$Session()
sess$list_devices()


# Loading sample dataset (100mb images). As is common in curated datasets, it is already cleanly organized into train and test subsets. 
cifar <- dataset_cifar10()


#TRAINING DATA
train_x<-cifar$train$x/255
#convert a vector class to binary class matrix
#converting the target variable to one hot encoded vectors using #keras inbuilt function 'to_categorical()
train_y<- to_categorical(cifar$train$y,num_classes = 10)

#TEST DATA
test_x<-cifar$test$x/255
test_y<-to_categorical(cifar$test$y,num_classes=10) 
#checking the dimensions
dim(train_x) 
cat("No of training samples\t",dim(train_x)[[1]],"\tNo of test samples\t",dim(test_x)[[1]])
```

<br>
<br>

# Building the Model


```r
#a linear stack of layers
model<-keras_model_sequential()
#configuring the Model
model %>%  
#defining a 2-D convolution layer: This layer creates a convolution kernel that is convolved with the layer input to produce a tensor of outputs. If use_bias is TRUE, a bias vector is created and added to the outputs Finally, if activation is not NULL, it is applied to the outputs as well. When using this layer as the first layer in a model, provide the keyword argument input_shape (list of integers, does not include the sample axis), e.g. input_shape=c(128, 128, 3) for 128x128 RGB pictures in data_format="channels_last".
# https://www.pyimagesearch.com/2018/12/31/keras-conv2d-and-convolutional-layers/
    
layer_conv_2d(filter=32,kernel_size=c(3,3),padding="same", input_shape=c(32,32,3) ) %>%  
layer_activation("relu") %>%  
#another 2-D convolution layer
  
layer_conv_2d(filter=32 ,kernel_size=c(3,3))  %>%  layer_activation("relu") %>%
#Defining a Pooling layer which reduces the dimensions of the #features map and reduces the computational complexity of the model
layer_max_pooling_2d(pool_size=c(2,2)) %>%  
#dropout layer to avoid overfitting
layer_dropout(0.25) %>%
layer_conv_2d(filter=32 , kernel_size=c(3,3),padding="same") %>% layer_activation("relu") %>%  layer_conv_2d(filter=32,kernel_size=c(3,3) ) %>%  layer_activation("relu") %>%  
layer_max_pooling_2d(pool_size=c(2,2)) %>%  
layer_dropout(0.25) %>%
#flatten the input  
layer_flatten() %>%  
layer_dense(512) %>%  
layer_activation("relu") %>%  
layer_dropout(0.5) %>%  
#output layer-10 classes-10 units  
layer_dense(10) %>%  
#applying softmax nonlinear activation function to the output layer #to calculate cross-entropy  
layer_activation("softmax") 
#for computing Probabilities of classes-"logit(log probabilities)
```

<br>
<br>

## Model Optimizer

```r
#defining the type of optimizer-ADAM-Adaptive Momentum Estimation
opt<-optimizer_adam( lr= 0.0001 , decay = 1e-6 )
#lr-learning rate , decay - learning rate decay over each update

model %>%
 compile(loss="categorical_crossentropy",
 optimizer=opt,metrics = "accuracy")
#Summary of the Model and its Architecture
summary(model)



# sink("Model_Summary.txt")
# print(summary(model))
# sink()
```

<img src="Model_Summary.png" alt="drawing" width="600" height="600"/>

<br>
<br>

## Model Training

```r
#https://github.com/tensorflow/tensorflow/issues/6698
# reticulate::physical_devices = tf.config.list_physical_devices('GPU')
# reticulate::tf.config.experimental.set_memory_growth(physical_devices[0], True)


#TRAINING PROCESS OF THE MODEL
data_augmentation <- TRUE  
if(!data_augmentation) {  
model %>% fit( train_x,train_y ,batch_size=32,
               epochs=80,validation_data = list(test_x, test_y),
               shuffle=TRUE)
} else {  
#Generating images
  
gen_images <- image_data_generator(featurewise_center = TRUE,
      featurewise_std_normalization = TRUE,
      rotation_range = 20,
      width_shift_range = 0.30,
      height_shift_range = 0.30,
      horizontal_flip = TRUE  )
#Fit image data generator internal statistics to sample data
gen_images %>% fit_image_data_generator(train_x)
#Generates batches of augmented/normalized data from image data and #labels to visually see the generated images by the Model
model %>% fit_generator(
     flow_images_from_data(train_x, train_y,gen_images,
     batch_size=32,save_to_dir="C:/.../R Projects/Keras/Keras/Test"),
     steps_per_epoch=as.integer(50000/32),epochs = 80,
     validation_data = list(test_x, test_y) )
}
#use save_to_dir argument to specify the directory to save the #images generated by the Model and to visually check the Model's #output and ability to classify images.
```

<br>
<br>

# Results

Initial attempt results in an accuracy of under 70%, leaving room for [improvement](https://appliedmachinelearning.blog/2018/03/24/achieving-90-accuracy-in-object-recognition-task-on-cifar-10-dataset-with-keras-convolutional-neural-networks/) for future attempts! I'll look to gain deeper understanding of keras, tensorflow, and deep learning methods and continue to iterate this work over the coming weeks.
<br>
<br>
<img src="cifar_accuracy.png" alt="drawing" width="600" height="600"/>
<br>
<br>

***

