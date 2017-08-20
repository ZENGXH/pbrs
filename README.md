# Physically Based Rendering for Indoor Scene Understanding

## Introduction:

This package includes the whole pipeline for generating physically based rendering synthetic data for indoor scene understanding. We also provide download links for most of the intermediate and final results. Please refer to the project webpage (http://pbrs.cs.princeton.edu) for more details.

## Citations:
If you use any part of the code or generated data, please cite the following paper:


	@article{zhang2016physically,
	  title={Physically-Based Rendering for Indoor Scene Understanding Using Convolutional Neural Networks},
	  author={Zhang, Yinda and Song, Shuran and Yumer, Ersin and Savva, Manolis and Lee, Joon-Young and Jin, Hailin and Funkhouser, Thomas},
	  journal={The IEEE Conference on Computer Vision and Pattern Recognition (CVPR)},
	  year={2017}
	}


## Dependency

To fully reproduce the data, you need several softwares and datasets listed below. Please check the links for their latest updates. We include a copy of them for consistency issue.
- [SUNCG Dataset](http://suncg.cs.princeton.edu). It provides the raw scene models required to render data from. These raw data grant you more flexibility to interact with the scene. You need to sign this [agreement form](http://suncg.cs.princeton.edu/form.pdf) in order to get full access of the dataset.
- [Gaps](https://github.com/shurans/SUNCGtoolbox/tree/master/gaps). The C++ parser for SUNCG dataset. It provides apps to generate camera viewpoints, scene obj models, and produce a variety of ground truth.
- [Mitsuba](http://www.mitsuba-renderer.org/). The physically based rendering engine.

These dependencies are all fairly easy to download and install. Alternatively, we provide you precomputed camera viewpoints and the fixed OBJ models and the serialized files ready for rendering.

## Quick start
1. Download modified xml and serialized file ready for rendering using Mitsuba. Since these data contain essential information from SUNCG, please send an email to me (yindazATcsDOTprincetonDOTedu) with a signed [agreement form](http://suncg.cs.princeton.edu/form.pdf) for the download link. Unzip these files to [`./rendering/projects_serialize`](./rendering/projects_serialize).
2. Download [precomputed camera views](http://pbrs.cs.princeton.edu/pbrs_release/data/camera_v2.zip). Unzip these file to [`./rendering/projects_camera`](./rendering/projects_camera)
3. Install Mitsuba (with python support).
4. Modify [`config.m`](./matlab/config.m) to load project list with all scenes, and run
```
addpath(genpath('matlab'));
config;
script_mitsuba_render;
system(sprintf('sh %s', [script_path 'cmd_mtsb_render/cmd_1.sh']));
```

Alternatively, you could check [`demo.m`](demo.m) for more details.

## Pipeline

### Generate camera
We use the `scn2cam` gaps to generate the scene view camera. This app brute force searches location and viewing direction in the scene and find those those camera views containing multiple objects covering considerable portion of pixels. It may take a long while to compute camera poses for all scenes. Alternatively, you can download the [precomputed camera views](http://pbrs.cs.princeton.edu/pbrs_release/data/camera_v2.zip).

From now on, we will use the following camera intrinsic:
```
517.97 0      320
0      517.97 240
0      0      1 
```
To generate camera viewpoints:
```
cd ./planner5d/house/[project_id]/
./gaps/bin/x86_64/scn2cam house.json ./projects_camera/[project_id]/room_camera.txt  -create_room_cameras -output_nodes ./rendering/projects_camera/[project_id]/room_camera_node.txt -output_camera_names ./rendering/projects_camera/[project_id]//room_camera_name.txt -categories ./util_data/ModelCategoryMappingNewActive.csv -xfov 0.553400 -mesa
```

We provide matlab script [`matlab/script_generate_camera.m`](matlab/script_generate_camera.m) to generate batch file for commands of all scenes. It allows to generate multiple batch files that could be run in parallel. 

Some rooms do not have any source of lighting, e.g. door, window, lamp. We use the SUNCG [online query engine](http://suncg.cs.princeton.edu/#)->Explore to get the list of such rooms, and then filter out cameras generated in these rooms according to the camera names. In short, you can retrieve the list of bad room with this url:
```
http://visiongpu.cs.princeton.edu:8498/solr/rooms/select?fl=id&indent=on&q=!modelCats:chandelier%20AND%20!modelCats:table_lamp%20AND%20!modelCats:floor_lamps%20AND%20!modelCats:wall_lamp%20AND%20!modelCats:window%20AND%20!modelCats:door%20AND%20!modelCats:garage_door&wt=csv&rows=100000
```

We have provided this list in util_data/bad_room.txt. Call [`matlab/filter_bad_camera.m`](matlab/filter_bad_camera.m) generates a file with 0/1 indicating bad or good cameras.


### Prepare 3D models
Mitsuba uses serialized file for 3D models. The serialized file also allow us to edit materials and geometry in a flexible manner. Follow the step below to generate serialized files and templates ready for rendering. Alternatively, we provide these files for downloading. Please send an email to me (yindazATcsDOTprincetonDOTedu) with a signed [agreement form](http://suncg.cs.princeton.edu/form.pdf) for the download link.

To generate serialized file, we first generate an obj file for each scene. Gaps provides an app `scn2scn` to generate whole scene obj model. However, the walls are represented by single planar surface, and rendering with which may cause light leakage in room corners. As such, we generate whole room obj models, i.e. one model for each house, ready to use. 
```
cd ./rendering/projects_serialize/[project_id]
wget http://link_require_agreement/[project_id].obj.xz
xz -kd [project_id]
```

The obj files are then converted to serialized file by calling the app `mtsimport` in Mitsuba. Note that this step requires the `texture` folder of SUNCG in [`./rendering/`](./rendering).
```
mv [project_id].obj main.obj
mtsimport main.obj main.xml # make sure collada is found when installing with scons, otherwise there will be no mtsimport
```

[`matlab/script_generate_serialized.m`](matlab/script_generate_serialized.m) generates batch command to quickly process all scenes.

This command generates a `main.serialized` encoding the geometry and texture, and a `main.xml` which indicates the association between two and additional materials. We modify the xml file to fix material, geometry, and lighing problem. See [`matlab/convert_mtsb_template.m`](matlab/convert_mtsb_template.m) for how to modify the xml. It generates a new xml file ready to be used by Mitsuba for rendering.


### Physically-based-rendering
The images can be rendered by:
```
python mitsuba_render.py -s ./planner5d/projects_serialized/[project_id]/main_template_color.xml -c ./projects_camera/[project_id]/room_camera.txt -o ./projects_render/[project_id]/ -g ./projects_camera/[project_id]/room_camera_good.txt
cd ./projects_render/[project_id]/
mtsutil tonemap -p 0.8,0.2 *.rgbe
```

[`matlab/script_mitsuba_render.m`](matlab/script_mitsuba_render.m) generates batch commands to render all the scenes.

### Ground truth 
We use Gaps to generate ground truth. 
```
cd ./planner5d/house/[project_id]/
./gaps/bin/x86_64/scn2img house.json ./projects_camera/[project_id]/room_camera.txt ./projects_render/[project_id]/ -categories ./util_data/ModelCategoryMappingNewActive.csv -capture_color_images -capture_depth_images -capture_normal_images -capture_node_images -width 640 -height 480 -headlight -mesa
```

This command will generate:
- `OpenGL color rendering`
- `Depth image`: 16-bit PNG, in millimeter.
- `Surface normal`: x, y, z channel, 16-bit PNG with 0-65535 mapping to -1 to 1, in world coordinate.
- `Node image`: instance segmentation map. Each id represent an entity in the scene. The room_camera_node.txt tells the entity each id represents.

We provide code to generate more useful ground truth with these raw ground truth
- `Surface normal` in camera coordinate: check `matlab/convert_camera_normal.m`
- `Instance boundary`: check `matlab/convert_instance_boundary.m`
