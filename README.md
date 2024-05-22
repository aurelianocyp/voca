

## Set-up

The code uses Python 3.7.16 and it was tested on tensorflow-gpu 1.15.2. 3090
```
sudo apt install ffmpeg
git clone https://github.com/aurelianocyp/voca.git
pip install -r requirements.txt
```

Install mesh processing libraries from [MPI-IS/mesh](https://github.com/MPI-IS/mesh) within the virtual environment.
- sudo apt update
- sudo apt-get install libboost-dev
- git clone https://github.com/MPI-IS/mesh.git
- cd mesh
- python -m pip install pip==22.2.1
- BOOST_INCLUDE_DIRS=/path/to/boost/include make all
- make tests
- 如果test时输出为OK (skipped=5)，应该就行了
- 换了conda环境后要重新libboost-dev
- 最好把mesh的requirement里全注释掉

## Data

#### Data to run the demo 

Download the trained VOCA model, audio sequences, and template meshes from [MPI-IS/VOCA](https://voca.is.tue.mpg.de).<br/>
Download FLAME model from [MPI-IS/FLAME](http://flame.is.tue.mpg.de/).<br/> 将这里面的output_graph.pb放入ds graph文件夹
Download the trained DeepSpeech model (v0.1.0) from [Mozilla/DeepSpeech](https://github.com/mozilla/DeepSpeech/releases/tag/v0.1.0) (i.e. deepspeech-0.1.0-models.tar.gz).

#### Data used to train VOCA

VOCA is trained on VOCASET, a unique 4D face dataset with about 29 minutes of 4D scans captured at 60 fps and synchronized audio from 12 speakers that can be downloaded at [MPI-IS/VOCASET](https://voca.is.tue.mpg.de). 

Training subjects:
```
FaceTalk_170728_03272_TA, FaceTalk_170904_00128_TA, FaceTalk_170725_00137_TA, FaceTalk_170915_00223_TA, FaceTalk_170811_03274_TA, FaceTalk_170913_03279_TA, FaceTalk_170904_03276_TA, FaceTalk_170912_03278_TA
```
This is also the order of the subjects for the one-hot-encoding (i.e. FaceTalk_170728_03272_TA: 0, FaceTalk_170904_00128_TA: 1, ...)

Validation subjects:
```
FaceTalk_170811_03275_TA, FaceTalk_170908_03277_TA
```

Test subjects:
```
FaceTalk_170809_00138_TA, FaceTalk_170731_00024_TA 
```

## Demo

We provide demos to
1) synthesize a character animation given an speech signal (VOCA),
2) add eye blinks, alter identity dependent face shape and head pose of an animation sequence using FLAME, and
3) generate templates (e.g. by sampling the [FLAME](http://flame.is.tue.mpg.de/) identity shape space, or by reconstructing a template from an image using [RingNet](https://github.com/soubhiksanyal/RingNet) that can be animated with VOCA.)

##### VOCA output

This demo runs VOCA, which outputs the animation meshes given audio sequences, and renders the animation sequence to a video.
```
python run_voca.py --tf_model_fname './model/gstep_52280.model' --ds_fname './ds_graph/output_graph.pb' --audio_fname './audio/test_sentence.wav' --template_fname './template/FLAME_sample.ply' --condition_idx 3 --out_path './animation_output'
```

To run VOCA and visualize the meshes with a pre-defined texture (obtained by fitting FLAME to an image using [TF_FLAME](https://github.com/TimoBolkart/TF_FLAME)), run:
```
python run_voca.py --tf_model_fname './model/gstep_52280.model' --ds_fname './ds_graph/output_graph.pb' --audio_fname './audio/test_sentence.wav' --template_fname './template/FLAME_sample.ply' --condition_idx 3 --uv_template_fname './template/texture_mesh.obj' --texture_img_fname './template/texture_mesh.png' --out_path './animation_output_textured'
```

By default, running the demo uses pyrender to render the sequence to a video, however this causes problems for certain configurations (e.g. if running the code remotely). In this case try running the demo with an additional flag ```--visualize False``` to disable the visualization. The animated meshes are then still stored to the output directory and can be viewed or rendered with another tool. condition应该是说话风格而已。要更换人的三维模型换掉flame sample就行。

##### Edit VOCA output

VOCA outputs meshes in FLAME topology in "zero pose". This allows to edit the output sequences by varying the FLAME model parameters. These demos shows how to use FLAME to add eye blinks, edit the identity dependent face shape, or head pose of a VOCA animation sequence.

Add eye blinks (FLAME2019 only):
```
python edit_sequences.py --source_path './animation_output/meshes' --out_path './FLAME_eye_blink' --flame_model_path  './flame/generic_model.pkl' --mode blink --num_blinks 2 --blink_duration 15
```
Please not that this only demonstrates how to use FLAME to manipulate the eyelids to blink. The output motion does not resemble a true eye blink. This demo works with the FLAME2019 model only.

Edit identity-dependent shape:
```
python edit_sequences.py --source_path './animation_output/meshes' --out_path './FLAME_variation_shape' --flame_model_path  './flame/generic_model.pkl' --mode shape --index 0 --max_variation 3
```

Edit head pose:
```
python edit_sequences.py --source_path './animation_output/meshes' --out_path './FLAME_variation_pose' --flame_model_path  './flame/generic_model.pkl' --mode pose --index 3 --max_variation 0.52
```

##### Render sequence

This demo renders an animation sequence to a video.
```
python visualize_sequence.py --sequence_path './FLAME_eye_blink/meshes' --audio_fname './audio/test_sentence.wav' --out_path './FLAME_eye_blink'
python visualize_sequence.py --sequence_path './FLAME_variation_shape/meshes' --audio_fname './audio/test_sentence.wav' --out_path './FLAME_variation_shape'
python visualize_sequence.py --sequence_path './FLAME_variation_pose/meshes' --audio_fname './audio/test_sentence.wav' --out_path './FLAME_variation_pose'
```
To visualize the sequences with a pre-defined texture, additionally specify the flags ```--uv_template_fname``` and ```--texture_img_fname``` as done for the run_voca demo.

##### Compute FLAME parameters

VOCA outputs meshes in FLAME topology in "zero pose". This demo shows how to compute the FLAME paramters for such a sequence. 
```
python compute_FLAME_params.py --source_path './animation_output/meshes' --params_fname './FLAME_parameters/params.npy' --flame_model_path  './flame/generic_model.pkl' --template_fname './template/FLAME_sample.ply' 
```
The ```--template_fname``` must specify the template provided to VOCA to generate the sequence. 

To reconstruct the FLAME meshes from the sequence paramters back:
```
python compute_FLAME_params.py --params_fname './FLAME_parameters/params.npy' --flame_model_path  './flame/generic_model.pkl' --out_path './FLAME_parameters/meshes' 
```


##### Sample template

VOCA animates static templates in FLAME topology. Such templates can be obtained by fitting FLAME to scans, images, or by sampling the FLAME shape space. This demo randomly samples the FLAME identity shape space to generate new templates.
```
python sample_templates.py --flame_model_path './flame/generic_model.pkl' --num_samples 1 --out_path './template'
```

##### Reconstruct template from images

[RingNet](https://ringnet.is.tue.mpg.de/) is a framework to fully automatically reconstruct 3D meshes in FLAME topology from an image. After removing effects of pose and expression, the RingNet output mesh can be used as VOCA template. Please see the RingNet [demo](https://github.com/soubhiksanyal/RingNet) on how to reconstruct a 3D mesh from an image with neutralized pose and expression.

## Training

We provide code to train a VOCA model. Prior to training, run the VOCA output demo, as the training shares the requirements.
Additionally, download the VOCA training data from [MPI-IS/VOCA](https://voca.is.tue.mpg.de).<br/>

The training code requires a config file containing all model training parameters. To create a config file, run
```
python config_parser.py
```

To start training, run
```
python run_training.py
```

To visualize the training progress, run
```
tensorboard --logdir='./training/summaries/' --port 6006
```
This generates a [link](http://localhost:6006/) on the command line.  Open the link with a web browser to show the visualization.

## Known issues

If you get an error like
```
ModuleNotFoundError: No module named 'psbody'
```
please check if the [MPI-IS/mesh](https://github.com/MPI-IS/mesh) is successfully installed within the virtual environment.


## 记录
要生成不带纹理的视频，删掉纹理图片即可。但是生成视频会闪烁。带纹理的也闪烁，只不过没那么明显









