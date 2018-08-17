# Cancercell_detection
{
 "cells": [
  {
   "cell_type": "code",
   "execution_count": 75,
   "metadata": {},
   "outputs": [
    {
     "ename": "SyntaxError",
     "evalue": "(unicode error) 'unicodeescape' codec can't decode bytes in position 2-3: truncated \\UXXXXXXXX escape (<ipython-input-75-ea0d6daeaa76>, line 7)",
     "output_type": "error",
     "traceback": [
      "\u001b[1;36m  File \u001b[1;32m\"<ipython-input-75-ea0d6daeaa76>\"\u001b[1;36m, line \u001b[1;32m7\u001b[0m\n\u001b[1;33m    labels_df=pd.read_csv('C:\\Users\\HP\\Desktop\\LungCancerDetection-master\\Stage1',index_col=0)\u001b[0m\n\u001b[1;37m                         ^\u001b[0m\n\u001b[1;31mSyntaxError\u001b[0m\u001b[1;31m:\u001b[0m (unicode error) 'unicodeescape' codec can't decode bytes in position 2-3: truncated \\UXXXXXXXX escape\n"
     ]
    }
   ],
   "source": [
    "import dicom\n",
    "\n",
    "import pandas as pd\n",
    "import os\n",
    "data_dir='E:/stage1/stage1_pre/'\n",
    "patients=os.listdir(data_dir)\n",
    "labels_df=pd.read_csv('C:\\Users\\HP\\Desktop\\LungCancerDetection-master\\Stage1',index_col=0)\n",
    "labels_df.head()\n"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 76,
   "metadata": {},
   "outputs": [
    {
     "data": {
      "text/html": [
       "<script>requirejs.config({paths: { 'plotly': ['https://cdn.plot.ly/plotly-latest.min']},});if(!window.Plotly) {{require(['plotly'],function(plotly) {window.Plotly=plotly;});}}</script>"
      ],
      "text/vnd.plotly.v1+html": [
       "<script>requirejs.config({paths: { 'plotly': ['https://cdn.plot.ly/plotly-latest.min']},});if(!window.Plotly) {{require(['plotly'],function(plotly) {window.Plotly=plotly;});}}</script>"
      ]
     },
     "metadata": {},
     "output_type": "display_data"
    }
   ],
   "source": [
    "import numpy as np\n",
    "import dicom\n",
    "import os\n",
    "import matplotlib.pyplot as plt\n",
    "from glob import glob\n",
    "from mpl_toolkits.mplot3d.art3d import Poly3DCollection\n",
    "import scipy.ndimage\n",
    "from skimage import morphology\n",
    "from skimage import measure\n",
    "from skimage.transform import resize\n",
    "from sklearn.cluster import KMeans\n",
    "from plotly import __version__\n",
    "from plotly.offline import download_plotlyjs, init_notebook_mode, plot, iplot\n",
    "from plotly.tools import FigureFactory as FF\n",
    "from plotly.graph_objs import *\n",
    "init_notebook_mode(connected=True) \n",
    "\n",
    "\n",
    "IMG_PX_SIZE=150\n",
    "def load_scan(path):\n",
    "    print(\"load scan\")\n",
    "    slices = [dicom.read_file(path + '/' + s) for s in os.listdir(path)]\n",
    "    slices.sort(key = lambda x: int(x.ImagePositionPatient[2]))\n",
    "    \n",
    "    try:\n",
    "        slice_thickness = np.abs(slices[0].ImagePositionPatient[2] - slices[1].ImagePositionPatient[2])\n",
    "    except:\n",
    "        slice_thickness = np.abs(slices[0].SliceLocation - slices[1].SliceLocation)\n",
    "        \n",
    "    for s in slices:\n",
    "        s.SliceThickness = slice_thickness\n",
    "        \n",
    "    return slices\n",
    "import cv2\n",
    "import numpy as np\n",
    "\n",
    "\n",
    "def get_pixels_hu(scans):\n",
    "    print(\"getting HU fields\")\n",
    "    image = np.stack([s.pixel_array for s in scans])\n",
    "    image = image.astype(np.int16)\n",
    "\n",
    "    image[image == -2000] = 0\n",
    "    \n",
    "    # Convert to Hounsfield units (HU)\n",
    "    intercept = scans[0].RescaleIntercept\n",
    "    slope = scans[0].RescaleSlope\n",
    "    \n",
    "    if slope != 1:\n",
    "        image = slope * image.astype(np.float64)\n",
    "        image = image.astype(np.int16)\n",
    "        \n",
    "    image += np.int16(intercept)\n",
    "    \n",
    "    return np.array(image, dtype=np.int16)\n",
    "\n",
    "def make_lungmask(img, display=False):\n",
    "    \n",
    "    row_size= img.shape[0]\n",
    "    col_size = img.shape[1]\n",
    "    \n",
    "    mean = np.mean(img)\n",
    "    std = np.std(img)\n",
    "    img = img-mean\n",
    "    img = img/std\n",
    "    middle = img[int(col_size/5):int(col_size/5*4),int(row_size/5):int(row_size/5*4)] \n",
    "    mean = np.mean(middle)  \n",
    "    max = np.max(img)\n",
    "    min = np.min(img)\n",
    "    \n",
    "    img[img==max]=mean\n",
    "    img[img==min]=mean\n",
    "    \n",
    "    kmeans = KMeans(n_clusters=2).fit(np.reshape(middle,[np.prod(middle.shape),1]))\n",
    "    centers = sorted(kmeans.cluster_centers_.flatten())\n",
    "    threshold = np.mean(centers)\n",
    "    thresh_img = np.where(img<threshold,1.0,0.0)  \n",
    "\n",
    "    \n",
    "    eroded = morphology.erosion(thresh_img,np.ones([3,3]))\n",
    "    dilation = morphology.dilation(eroded,np.ones([8,8]))\n",
    "\n",
    "    labels = measure.label(dilation) \n",
    "    label_vals = np.unique(labels)\n",
    "    regions = measure.regionprops(labels)\n",
    "    good_labels = []\n",
    "    for prop in regions:\n",
    "        B = prop.bbox\n",
    "        if B[2]-B[0]<row_size/10*9 and B[3]-B[1]<col_size/10*9 and B[0]>row_size/5 and B[2]<col_size/5*4:\n",
    "            good_labels.append(prop.label)\n",
    "    mask = np.ndarray([row_size,col_size],dtype=np.int8)\n",
    "    mask[:] = 0\n",
    "\n",
    "   \n",
    "    for N in good_labels:\n",
    "        mask = mask + np.where(labels==N,1,0)\n",
    "    mask = morphology.dilation(mask,np.ones([10,10])) # one last dilation\n",
    "\n",
    "    if (display):\n",
    "        fig, ax = plt.subplots(3, 2, figsize=[12, 12])\n",
    "        ax[0, 0].set_title(\"Original\")\n",
    "        ax[0, 0].imshow(img, cmap='gray')\n",
    "        ax[0, 0].axis('off')\n",
    "        ax[0, 1].set_title(\"Threshold\")\n",
    "        ax[0, 1].imshow(thresh_img, cmap='gray')\n",
    "        ax[0, 1].axis('off')\n",
    "        ax[1, 0].set_title(\"After Erosion and Dilation\")\n",
    "        ax[1, 0].imshow(dilation, cmap='gray')\n",
    "        ax[1, 0].axis('off')\n",
    "        ax[1, 1].set_title(\"Color Labels\")\n",
    "        ax[1, 1].imshow(labels)\n",
    "        ax[1, 1].axis('off')\n",
    "        ax[2, 0].set_title(\"Final Mask\")\n",
    "        ax[2, 0].imshow(mask, cmap='gray')\n",
    "        ax[2, 0].axis('off')\n",
    "        ax[2, 1].set_title(\"Apply Mask on Original\")\n",
    "        ax[2, 1].imshow(mask*img, cmap='gray')\n",
    "        ax[2, 1].axis('off')\n",
    "        \n",
    "        plt.show()\n",
    "    return mask*img\n",
    "\n",
    "\n"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "id=0\n",
    "for patient in patients:\n",
    "    path = data_dir + patient\n",
    "    preprocess(id,path)\n",
    "    id=id+1"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "import matplotlib.pyplot as plt\n",
    "\n",
    "for patient in patients[:1]:\n",
    "    label = labels_df.get_value(patient, 'cancer')\n",
    "    path = data_dir + patient\n",
    "    slices = [dicom.read_file(path + '/' + s) for s in os.listdir(path)]\n",
    "    slices.sort(key = lambda x: int(x.ImagePositionPatient[2]))\n",
    "    \n",
    "       plt.imshow(slices[0].pixel_array)\n",
    "    plt.show()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 5,
   "metadata": {},
   "outputs": [],
   "source": [
    "id=0"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "import numpy as np\n",
    "import pandas as pd\n",
    "import dicom\n",
    "import os\n",
    "import matplotlib.pyplot as plt\n",
    "import cv2\n",
    "import math\n",
    "\n",
    "IMG_SIZE_PX = 50\n",
    "SLICE_COUNT = 20\n",
    "\n",
    "def chunks(l, n):\n",
    "   \n",
    "    for i in range(0, len(l), n):\n",
    "        yield l[i:i + n]\n",
    "\n",
    "\n",
    "def mean(a):\n",
    "    return sum(a) / len(a)\n",
    "\n",
    "\n",
    "def process_data(num,patient,labels_df,img_px_size=50, hm_slices=20, visualize=False):\n",
    "    id=num\n",
    "    label = labels_df.get_value(patient, 'cancer')\n",
    "    path = data_dir + patient\n",
    "    slices = [dicom.read_file(path + '/' + s) for s in os.listdir(path)]\n",
    "    slices.sort(key = lambda x: int(x.ImagePositionPatient[2]))\n",
    "    file_used= \"E:/stage1/test_prep/\"+\"finalimages_%d.npy\" % id\n",
    "    imgs_to_process = np.load(file_used).astype(np.uint16) \n",
    "\n",
    "    for num,each_slice in enumerate(slices):\n",
    "        np.copyto(each_slice.pixel_array, imgs_to_process[num])\n",
    "    \n",
    "\n",
    "    new_slices = []\n",
    "    slices = [cv2.resize(np.array(each_slice.pixel_array),(img_px_size,img_px_size)) for each_slice in slices]\n",
    "    \n",
    "    chunk_sizes = math.ceil(len(slices) / hm_slices)\n",
    "    for slice_chunk in chunks(slices, chunk_sizes):\n",
    "        slice_chunk = list(map(mean, zip(*slice_chunk)))\n",
    "        new_slices.append(slice_chunk)\n",
    "\n",
    "    if len(new_slices) == hm_slices-1:\n",
    "        new_slices.append(new_slices[-1])\n",
    "\n",
    "    if len(new_slices) == hm_slices-2:\n",
    "        new_slices.append(new_slices[-1])\n",
    "        new_slices.append(new_slices[-1])\n",
    "\n",
    "    if len(new_slices) == hm_slices+2:\n",
    "        new_val = list(map(mean, zip(*[new_slices[hm_slices-1],new_slices[hm_slices],])))\n",
    "        del new_slices[hm_slices]\n",
    "        new_slices[hm_slices-1] = new_val\n",
    "        \n",
    "    if len(new_slices) == hm_slices+1:\n",
    "        new_val = list(map(mean, zip(*[new_slices[hm_slices-1],new_slices[hm_slices],])))\n",
    "        del new_slices[hm_slices]\n",
    "        new_slices[hm_slices-1] = new_val\n",
    "\n",
    "    if visualize:\n",
    "        fig = plt.figure()\n",
    "        for num,each_slice in enumerate(new_slices):\n",
    "            y = fig.add_subplot(4,5,num+1)\n",
    "            y.imshow(each_slice, cmap='gray')\n",
    "        plt.show()\n",
    "\n",
    "    if label == 1: label=np.array([0,1])\n",
    "    elif label == 0: label=np.array([1,0])\n",
    "        \n",
    "    return np.array(new_slices),label\n",
    "\n",
    "\n",
    "data_dir='E:/stage1/Testing/'\n",
    "\n",
    "\n",
    "patients = os.listdir(data_dir)\n",
    "labels = pd.read_csv('stage1_labels.csv/stage1_labels.csv', index_col=0)\n",
    "\n",
    "much_data = []\n",
    "for num,patient in enumerate(patients):\n",
    "    print(num)\n",
    "    try:\n",
    "        \n",
    "        img_data,label = process_data(num,patient,labels,img_px_size=IMG_SIZE_PX, hm_slices=SLICE_COUNT)\n",
    "        #print(img_data.shape,label)\n",
    "        much_data.append([img_data,label])\n",
    "    except KeyError as e:\n",
    "        print('This is unlabeled data!')\n",
    "\n",
    "\n",
    "np.save('muchdatatest-{}-{}-{}.npy'.format(IMG_SIZE_PX,IMG_SIZE_PX,SLICE_COUNT), much_data)\n"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 29,
   "metadata": {},
   "outputs": [
    {
     "data": {
      "text/plain": [
       "'E:/stage1/stage1_pre/'"
      ]
     },
     "execution_count": 29,
     "metadata": {},
     "output_type": "execute_result"
    }
   ],
   "source": [
    "data_dir"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 51,
   "metadata": {},
   "outputs": [
    {
     "ename": "NameError",
     "evalue": "name 'num' is not defined",
     "output_type": "error",
     "traceback": [
      "\u001b[1;31m---------------------------------------------------------------------------\u001b[0m",
      "\u001b[1;31mNameError\u001b[0m                                 Traceback (most recent call last)",
      "\u001b[1;32m<ipython-input-51-c774dac2b598>\u001b[0m in \u001b[0;36m<module>\u001b[1;34m()\u001b[0m\n\u001b[1;32m----> 1\u001b[1;33m \u001b[0mnum\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[0m",
      "\u001b[1;31mNameError\u001b[0m: name 'num' is not defined"
     ]
    }
   ],
   "source": [
    "num"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 77,
   "metadata": {},
   "outputs": [],
   "source": [
    "import cv2"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 10,
   "metadata": {},
   "outputs": [],
   "source": [
    "import numpy as np\n",
    "IMG_PX_SIXE=150"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 39,
   "metadata": {},
   "outputs": [],
   "source": [
    "import tensorflow as tf\n"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 78,
   "metadata": {},
   "outputs": [],
   "source": [
    "import tensorflow as tf\n",
    "import numpy as np\n",
    "\n",
    "IMG_SIZE_PX = 50\n",
    "SLICE_COUNT = 20\n",
    "\n",
    "n_classes = 2\n",
    "batch_size = 10\n",
    "\n",
    "x = tf.placeholder('float')\n",
    "y = tf.placeholder('float')\n",
    "\n",
    "keep_rate = 0.8\n"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 13,
   "metadata": {},
   "outputs": [],
   "source": [
    "def conv3d(x, W):\n",
    "    return tf.nn.conv3d(x, W, strides=[1,1,1,1,1], padding='SAME')\n",
    "\n",
    "def maxpool3d(x):\n",
    "    #                        size of window         movement of window as you slide about\n",
    "    return tf.nn.max_pool3d(x, ksize=[1,2,2,2,1], strides=[1,2,2,2,1], padding='SAME')\n",
    "def load_neural_net():\n",
    "   \n",
    "    x=5\n",
    "    return x\n",
    "def printlabel(d,patient):\n",
    "        label = labels_df.get_value(patient, 'cancer')\n",
    "        path = data_dir + patient\n",
    "        slices = [dicom.read_file(path + '/' + s) for s in os.listdir(path)]\n",
    "        slices.sort(key = lambda x: int(x.ImagePositionPatient[2]))\n",
    "        print(\"Cancer\")\n",
    "        print(label)\n",
    "        \n",
    "        plt.imshow(slices[0].pixel_array)\n",
    "        plt.show()\n",
    "\n",
    "    "
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 14,
   "metadata": {},
   "outputs": [],
   "source": [
    "def convolutional_neural_network(x):\n",
    "    #                \n",
    "    weights = {'W_conv1':tf.Variable(tf.random_normal([3,3,3,1,32])),\n",
    "               \n",
    "               'W_conv2':tf.Variable(tf.random_normal([3,3,3,32,64])),\n",
    "              \n",
    "               'W_fc':tf.Variable(tf.random_normal([54080,1024])),\n",
    "               'out':tf.Variable(tf.random_normal([1024, n_classes]))}\n",
    "\n",
    "    biases = {'b_conv1':tf.Variable(tf.random_normal([32])),\n",
    "               'b_conv2':tf.Variable(tf.random_normal([64])),\n",
    "               'b_fc':tf.Variable(tf.random_normal([1024])),\n",
    "               'out':tf.Variable(tf.random_normal([n_classes]))}\n",
    "\n",
    "   \n",
    "    x = tf.reshape(x, shape=[-1, IMG_SIZE_PX, IMG_SIZE_PX, SLICE_COUNT, 1])\n",
    "\n",
    "    conv1 = tf.nn.relu(conv3d(x, weights['W_conv1']) + biases['b_conv1'])\n",
    "    conv1 = maxpool3d(conv1)\n",
    "\n",
    "\n",
    "    conv2 = tf.nn.relu(conv3d(conv1, weights['W_conv2']) + biases['b_conv2'])\n",
    "    conv2 = maxpool3d(conv2)\n",
    "\n",
    "    fc = tf.reshape(conv2,[-1, 54080])\n",
    "    fc = tf.nn.relu(tf.matmul(fc, weights['W_fc'])+biases['b_fc'])\n",
    "    fc = tf.nn.dropout(fc, keep_rate)\n",
    "\n",
    "    output = tf.matmul(fc, weights['out'])+biases['out']\n",
    "\n",
    "    return output\n"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 79,
   "metadata": {},
   "outputs": [
    {
     "ename": "FileNotFoundError",
     "evalue": "[Errno 2] No such file or directory: 'muchdatayass-50-50-20.npy'",
     "output_type": "error",
     "traceback": [
      "\u001b[1;31m---------------------------------------------------------------------------\u001b[0m",
      "\u001b[1;31mFileNotFoundError\u001b[0m                         Traceback (most recent call last)",
      "\u001b[1;32m<ipython-input-79-7c9d1ea5f30e>\u001b[0m in \u001b[0;36m<module>\u001b[1;34m()\u001b[0m\n\u001b[1;32m----> 1\u001b[1;33m \u001b[0mmuch_data\u001b[0m \u001b[1;33m=\u001b[0m \u001b[0mnp\u001b[0m\u001b[1;33m.\u001b[0m\u001b[0mload\u001b[0m\u001b[1;33m(\u001b[0m\u001b[1;34m'muchdatayass-50-50-20.npy'\u001b[0m\u001b[1;33m)\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[0m\u001b[0;32m      2\u001b[0m \u001b[0mtrain_data\u001b[0m \u001b[1;33m=\u001b[0m \u001b[0mmuch_data\u001b[0m\u001b[1;33m[\u001b[0m\u001b[1;33m:\u001b[0m\u001b[1;33m-\u001b[0m\u001b[1;36m120\u001b[0m\u001b[1;33m]\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[0;32m      3\u001b[0m \u001b[0mvalidation_data\u001b[0m \u001b[1;33m=\u001b[0m \u001b[0mmuch_data\u001b[0m\u001b[1;33m[\u001b[0m\u001b[1;33m-\u001b[0m\u001b[1;36m120\u001b[0m\u001b[1;33m:\u001b[0m\u001b[1;33m]\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[0;32m      4\u001b[0m \u001b[0msaver\u001b[0m\u001b[1;33m=\u001b[0m\u001b[0mtf\u001b[0m\u001b[1;33m.\u001b[0m\u001b[0mtrain\u001b[0m\u001b[1;33m.\u001b[0m\u001b[0mSaver\u001b[0m\u001b[1;33m(\u001b[0m\u001b[1;33m)\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n",
      "\u001b[1;32m~\\Anaconda3\\lib\\site-packages\\numpy\\lib\\npyio.py\u001b[0m in \u001b[0;36mload\u001b[1;34m(file, mmap_mode, allow_pickle, fix_imports, encoding)\u001b[0m\n\u001b[0;32m    370\u001b[0m     \u001b[0mown_fid\u001b[0m \u001b[1;33m=\u001b[0m \u001b[1;32mFalse\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[0;32m    371\u001b[0m     \u001b[1;32mif\u001b[0m \u001b[0misinstance\u001b[0m\u001b[1;33m(\u001b[0m\u001b[0mfile\u001b[0m\u001b[1;33m,\u001b[0m \u001b[0mbasestring\u001b[0m\u001b[1;33m)\u001b[0m\u001b[1;33m:\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[1;32m--> 372\u001b[1;33m         \u001b[0mfid\u001b[0m \u001b[1;33m=\u001b[0m \u001b[0mopen\u001b[0m\u001b[1;33m(\u001b[0m\u001b[0mfile\u001b[0m\u001b[1;33m,\u001b[0m \u001b[1;34m\"rb\"\u001b[0m\u001b[1;33m)\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[0m\u001b[0;32m    373\u001b[0m         \u001b[0mown_fid\u001b[0m \u001b[1;33m=\u001b[0m \u001b[1;32mTrue\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[0;32m    374\u001b[0m     \u001b[1;32melif\u001b[0m \u001b[0mis_pathlib_path\u001b[0m\u001b[1;33m(\u001b[0m\u001b[0mfile\u001b[0m\u001b[1;33m)\u001b[0m\u001b[1;33m:\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n",
      "\u001b[1;31mFileNotFoundError\u001b[0m: [Errno 2] No such file or directory: 'muchdatayass-50-50-20.npy'"
     ]
    }
   ],
   "source": [
    "much_data = np.load('muchdatayass-50-50-20.npy')\n",
    "train_data = much_data[:-120]\n",
    "validation_data = much_data[-120:]\n",
    "saver=tf.train.Saver()\n"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 63,
   "metadata": {},
   "outputs": [],
   "source": [
    "def train_neural_network(x):\n",
    "    prediction = convolutional_neural_network(x)\n",
    "   \n",
    "    cost = tf.reduce_mean( tf.nn.softmax_cross_entropy_with_logits(logits=prediction, labels=y) )\n",
    "\n",
    "    optimizer = tf.train.AdamOptimizer(learning_rate=1e-3).minimize(cost)\n",
    "\n",
    "    hm_epochs = 10\n",
    "    with tf.Session() as sess:\n",
    "        sess.run(tf.global_variables_initializer())        \n",
    "        successful_runs = 0\n",
    "        total_runs = 0\n",
    "\n",
    "        for epoch in range(hm_epochs):\n",
    "            epoch_loss = 0\n",
    "\n",
    "            for data in train_data:\n",
    "                total_runs += 1\n",
    "                try:\n",
    "                    X = data[0]\n",
    "                    Y = data[1]\n",
    "                    _, c = sess.run([optimizer, cost], feed_dict={x: X, y: Y})\n",
    "                    epoch_loss += c\n",
    "                    successful_runs += 1\n",
    "                except Exception as e:\n",
    "                    pass\n",
    "                \n",
    "            print('Epoch', epoch+1, 'completed out of',hm_epochs,'loss:',epoch_loss)\n",
    "\n",
    "            correct = tf.equal(tf.argmax(prediction, 1), tf.argmax(y, 1))\n",
    "            accuracy = tf.reduce_mean(tf.cast(correct, 'float'))\n",
    "\n",
    "            print('Accuracy:',accuracy.eval({x:[i[0] for i in validation_data], y:[i[1] for i in validation_data]}))            \n",
    "        \n",
    "\n",
    "        print('Done. Finishing accuracy:')\n",
    "        print('Accuracy:',accuracy.eval({x:[i[0] for i in validation_data], y:[i[1] for i in validation_data]}))\n",
    "\n",
    "        print('fitment percent:',successful_runs/total_runs)\n",
    "\n"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 59,
   "metadata": {},
   "outputs": [
    {
     "ename": "NameError",
     "evalue": "name 'labels_df' is not defined",
     "output_type": "error",
     "traceback": [
      "\u001b[1;31m---------------------------------------------------------------------------\u001b[0m",
      "\u001b[1;31mNameError\u001b[0m                                 Traceback (most recent call last)",
      "\u001b[1;32m<ipython-input-59-be4219597d18>\u001b[0m in \u001b[0;36m<module>\u001b[1;34m()\u001b[0m\n\u001b[1;32m----> 1\u001b[1;33m \u001b[0mlabels_df\u001b[0m\u001b[1;33m.\u001b[0m\u001b[0mcancer\u001b[0m\u001b[1;33m.\u001b[0m\u001b[0mvalue_counts\u001b[0m\u001b[1;33m(\u001b[0m\u001b[1;33m)\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[0m",
      "\u001b[1;31mNameError\u001b[0m: name 'labels_df' is not defined"
     ]
    }
   ],
   "source": [
    "labels_df.cancer.value_counts()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 56,
   "metadata": {},
   "outputs": [
    {
     "ename": "NameError",
     "evalue": "name 'labels_df' is not defined",
     "output_type": "error",
     "traceback": [
      "\u001b[1;31m---------------------------------------------------------------------------\u001b[0m",
      "\u001b[1;31mNameError\u001b[0m                                 Traceback (most recent call last)",
      "\u001b[1;32m<ipython-input-56-f8cafea3d7f1>\u001b[0m in \u001b[0;36m<module>\u001b[1;34m()\u001b[0m\n\u001b[1;32m----> 1\u001b[1;33m \u001b[0mlabels_df\u001b[0m\u001b[1;33m.\u001b[0m\u001b[0mix\u001b[0m\u001b[1;33m[\u001b[0m\u001b[1;33m-\u001b[0m\u001b[1;36m120\u001b[0m\u001b[1;33m:\u001b[0m\u001b[1;33m]\u001b[0m\u001b[1;33m.\u001b[0m\u001b[0mcancer\u001b[0m\u001b[1;33m.\u001b[0m\u001b[0mvalue_counts\u001b[0m\u001b[1;33m(\u001b[0m\u001b[1;33m)\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[0m",
      "\u001b[1;31mNameError\u001b[0m: name 'labels_df' is not defined"
     ]
    }
   ],
   "source": [
    "labels_df.ix[-120:].cancer.value_counts()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 80,
   "metadata": {},
   "outputs": [
    {
     "ename": "PermissionError",
     "evalue": "[WinError 21] The device is not ready: 'E:/stage1/testing/'",
     "output_type": "error",
     "traceback": [
      "\u001b[1;31m---------------------------------------------------------------------------\u001b[0m",
      "\u001b[1;31mPermissionError\u001b[0m                           Traceback (most recent call last)",
      "\u001b[1;32m<ipython-input-80-5d515b97c849>\u001b[0m in \u001b[0;36m<module>\u001b[1;34m()\u001b[0m\n\u001b[0;32m      1\u001b[0m \u001b[0mdata_dir\u001b[0m\u001b[1;33m=\u001b[0m\u001b[1;34m'E:/stage1/testing/'\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[1;32m----> 2\u001b[1;33m \u001b[0mpatients\u001b[0m\u001b[1;33m=\u001b[0m\u001b[0mos\u001b[0m\u001b[1;33m.\u001b[0m\u001b[0mlistdir\u001b[0m\u001b[1;33m(\u001b[0m\u001b[0mdata_dir\u001b[0m\u001b[1;33m)\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[0m",
      "\u001b[1;31mPermissionError\u001b[0m: [WinError 21] The device is not ready: 'E:/stage1/testing/'"
     ]
    }
   ],
   "source": [
    "data_dir='E:/stage1/testing/'\n",
    "patients=os.listdir(data_dir)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 60,
   "metadata": {},
   "outputs": [
    {
     "ename": "NameError",
     "evalue": "name 'patients' is not defined",
     "output_type": "error",
     "traceback": [
      "\u001b[1;31m---------------------------------------------------------------------------\u001b[0m",
      "\u001b[1;31mNameError\u001b[0m                                 Traceback (most recent call last)",
      "\u001b[1;32m<ipython-input-60-c914005de8b2>\u001b[0m in \u001b[0;36m<module>\u001b[1;34m()\u001b[0m\n\u001b[0;32m      1\u001b[0m \u001b[0mid\u001b[0m\u001b[1;33m=\u001b[0m\u001b[1;36m0\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[1;32m----> 2\u001b[1;33m \u001b[1;32mfor\u001b[0m \u001b[0mpatient\u001b[0m \u001b[1;32min\u001b[0m \u001b[0mpatients\u001b[0m\u001b[1;33m:\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[0m\u001b[0;32m      3\u001b[0m     \u001b[0mpath\u001b[0m \u001b[1;33m=\u001b[0m \u001b[0mdata_dir\u001b[0m \u001b[1;33m+\u001b[0m \u001b[0mpatient\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[0;32m      4\u001b[0m     \u001b[0mpreprocess\u001b[0m\u001b[1;33m(\u001b[0m\u001b[0mid\u001b[0m\u001b[1;33m,\u001b[0m\u001b[0mpath\u001b[0m\u001b[1;33m)\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[0;32m      5\u001b[0m     \u001b[0mid\u001b[0m\u001b[1;33m=\u001b[0m\u001b[0mid\u001b[0m\u001b[1;33m+\u001b[0m\u001b[1;36m1\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n",
      "\u001b[1;31mNameError\u001b[0m: name 'patients' is not defined"
     ]
    }
   ],
   "source": [
    "id=0\n",
    "for patient in patients:\n",
    "    path = data_dir + patient\n",
    "    preprocess(id,path)\n",
    "    id=id+1"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 43,
   "metadata": {},
   "outputs": [
    {
     "ename": "NameError",
     "evalue": "name 'patients' is not defined",
     "output_type": "error",
     "traceback": [
      "\u001b[1;31m---------------------------------------------------------------------------\u001b[0m",
      "\u001b[1;31mNameError\u001b[0m                                 Traceback (most recent call last)",
      "\u001b[1;32m<ipython-input-43-c914005de8b2>\u001b[0m in \u001b[0;36m<module>\u001b[1;34m()\u001b[0m\n\u001b[0;32m      1\u001b[0m \u001b[0mid\u001b[0m\u001b[1;33m=\u001b[0m\u001b[1;36m0\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[1;32m----> 2\u001b[1;33m \u001b[1;32mfor\u001b[0m \u001b[0mpatient\u001b[0m \u001b[1;32min\u001b[0m \u001b[0mpatients\u001b[0m\u001b[1;33m:\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[0m\u001b[0;32m      3\u001b[0m     \u001b[0mpath\u001b[0m \u001b[1;33m=\u001b[0m \u001b[0mdata_dir\u001b[0m \u001b[1;33m+\u001b[0m \u001b[0mpatient\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[0;32m      4\u001b[0m     \u001b[0mpreprocess\u001b[0m\u001b[1;33m(\u001b[0m\u001b[0mid\u001b[0m\u001b[1;33m,\u001b[0m\u001b[0mpath\u001b[0m\u001b[1;33m)\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[0;32m      5\u001b[0m     \u001b[0mid\u001b[0m\u001b[1;33m=\u001b[0m\u001b[0mid\u001b[0m\u001b[1;33m+\u001b[0m\u001b[1;36m1\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n",
      "\u001b[1;31mNameError\u001b[0m: name 'patients' is not defined"
     ]
    }
   ],
   "source": [
    "id=0\n",
    "for patient in patients:\n",
    "    path = data_dir + patient\n",
    "    preprocess(id,path)\n",
    "    id=id+1"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 81,
   "metadata": {},
   "outputs": [
    {
     "ename": "NameError",
     "evalue": "name 'patients' is not defined",
     "output_type": "error",
     "traceback": [
      "\u001b[1;31m---------------------------------------------------------------------------\u001b[0m",
      "\u001b[1;31mNameError\u001b[0m                                 Traceback (most recent call last)",
      "\u001b[1;32m<ipython-input-81-a9ee5eabf182>\u001b[0m in \u001b[0;36m<module>\u001b[1;34m()\u001b[0m\n\u001b[0;32m      1\u001b[0m \u001b[1;33m\u001b[0m\u001b[0m\n\u001b[0;32m      2\u001b[0m \u001b[1;33m\u001b[0m\u001b[0m\n\u001b[1;32m----> 3\u001b[1;33m \u001b[1;32mfor\u001b[0m \u001b[0mpatient\u001b[0m \u001b[1;32min\u001b[0m \u001b[0mpatients\u001b[0m\u001b[1;33m:\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[0m\u001b[0;32m      4\u001b[0m     \u001b[0msession\u001b[0m \u001b[1;33m=\u001b[0m\u001b[0mload_neural_net\u001b[0m\u001b[1;33m(\u001b[0m\u001b[1;33m)\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n\u001b[0;32m      5\u001b[0m     \u001b[0mprintlabel\u001b[0m\u001b[1;33m(\u001b[0m\u001b[0msession\u001b[0m\u001b[1;33m,\u001b[0m\u001b[0mpatient\u001b[0m\u001b[1;33m)\u001b[0m\u001b[1;33m\u001b[0m\u001b[0m\n",
      "\u001b[1;31mNameError\u001b[0m: name 'patients' is not defined"
     ]
    }
   ],
   "source": [
    "\n",
    "\n",
    "for patient in patients:\n",
    "    session =load_neural_net()\n",
    "    printlabel(session,patient)\n",
    "    "
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": []
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": []
  }
 ],
 "metadata": {
  "kernelspec": {
   "display_name": "Python 3",
   "language": "python",
   "name": "python3"
  },
  "language_info": {
   "codemirror_mode": {
    "name": "ipython",
    "version": 3
   },
   "file_extension": ".py",
   "mimetype": "text/x-python",
   "name": "python",
   "nbconvert_exporter": "python",
   "pygments_lexer": "ipython3",
   "version": "3.6.5"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 2
}

