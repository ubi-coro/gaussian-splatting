## Processing your own Scenes

Our COLMAP loaders expect the following dataset structure in the source path location:

```
<location>
|---images
|   |---<image 0>
|   |---<image 1>
|   |---...
|---sparse
    |---0
        |---cameras.bin
        |---images.bin
        |---points3D.bin
```

For rasterization, the camera models must be either a SIMPLE_PINHOLE or PINHOLE camera. We provide a converter script ```convert.py```, to extract undistorted images and SfM information from input images. Optionally, you can use ImageMagick to resize the undistorted images. This rescaling is similar to MipNeRF360, i.e., it creates images with 1/2, 1/4 and 1/8 the original resolution in corresponding folders. To use them, please first install a recent version of COLMAP (ideally CUDA-powered) and ImageMagick. Put the images you want to use in a directory ```<location>/input```.
```
<location>
|---input
    |---<image 0>
    |---<image 1>
    |---...
```
 If you have COLMAP and ImageMagick on your system path, you can simply run 
```shell
python convert.py -s <location> [--resize] #If not resizing, ImageMagick is not needed
```
Alternatively, you can use the optional parameters ```--colmap_executable``` and ```--magick_executable``` to point to the respective paths. Please note that on Windows, the executable should point to the COLMAP ```.bat``` file that takes care of setting the execution environment. Once done, ```<location>``` will contain the expected COLMAP data set structure with undistorted, resized input images, in addition to your original images and some temporary (distorted) data in the directory ```distorted```.

If you have your own COLMAP dataset without undistortion (e.g., using ```OPENCV``` camera), you can try to just run the last part of the script: Put the images in ```input``` and the COLMAP info in a subdirectory ```distorted```:
```
<location>
|---input
|   |---<image 0>
|   |---<image 1>
|   |---...
|---distorted
    |---database.db
    |---sparse
        |---0
            |---...
```
Then run 
```shell
python convert.py -s <location> --skip_matching [--resize] #If not resizing, ImageMagick is not needed
```

<details>
<summary><span style="font-weight: bold;">Command Line Arguments for convert.py</span></summary>

  |  |  |
  | :-- | :-- |
  | **--no_gpu** | Flag to avoid using GPU in COLMAP. |
  | **--skip_matching** | Flag to indicate that COLMAP info is available for images. |
  | **--source_path**<br>**-s** | Location of the inputs. |
  | **--camera** | Which camera model to use for the early matching steps, ```OPENCV``` by default. |
  | **--resize** | Flag for creating resized versions of input images. |
  | **--colmap_executable** | Path to the COLMAP executable (```.bat``` on Windows). |
  | **--magick_executable** | Path to the ImageMagick executable. |
</details>
<br>

## Training speed acceleration

We integrated the drop-in replacements from [Taming-3dgs](https://humansensinglab.github.io/taming-3dgs/)<sup>1</sup> with [fused ssim](https://github.com/rahul-goel/fused-ssim/tree/main) into the original codebase to speed up training times. Once installed, the accelerated rasterizer delivers a **$\times$ 1.6 training time speedup** using `--optimizer_type default` and a **$\times$ 2.7 training time speedup** using `--optimizer_type sparse_adam`.

To get faster training times you must first install the accelerated rasterizer to your environment:

```bash
pip uninstall diff-gaussian-rasterization -y
cd submodules/diff-gaussian-rasterization
rm -r build
git checkout 3dgs_accel
pip install .
```

Then you can add the following parameter to use the sparse adam optimizer when running `train.py`:

```bash
--optimizer_type sparse_adam
```

*Note that this custom rasterizer has a different behaviour than the original version, for more details on training times please see [stats for training times](results.md/#training-times-comparisons)*.

*1. Mallick and Goel, et al. ‘Taming 3DGS: High-Quality Radiance Fields with Limited Resources’. SIGGRAPH Asia 2024 Conference Papers, 2024, https://doi.org/10.1145/3680528.3687694, [github](https://github.com/humansensinglab/taming-3dgs)*


## Depth regularization

To have better reconstructed scenes we use depth maps as priors during optimization with each input images. It works best on untextured parts ex: roads and can remove floaters. Several papers have used similar ideas to improve various aspects of 3DGS; (e.g. [DepthRegularizedGS](https://robot0321.github.io/DepthRegGS/index.html), [SparseGS](https://formycat.github.io/SparseGS-Real-Time-360-Sparse-View-Synthesis-using-Gaussian-Splatting/), [DNGaussian](https://fictionarry.github.io/DNGaussian/)). The depth regularization we integrated is that used in our [Hierarchical 3DGS](https://repo-sam.inria.fr/fungraph/hierarchical-3d-gaussians/) paper, but applied to the original 3DGS; for some scenes (e.g., the DeepBlending scenes) it improves quality significantly; for others it either makes a small difference or can even be worse. For example results showing the potential benefit and statistics on quality please see here: [Stats for depth regularization](results.md).

When training on a synthetic dataset, depth maps can be produced and they do not require further processing to be used in our method.

For real world datasets depth maps should be generated for each input images, to generate them please do the following:
1. Clone [Depth Anything v2](https://github.com/DepthAnything/Depth-Anything-V2?tab=readme-ov-file#usage):
    ```
    git clone https://github.com/DepthAnything/Depth-Anything-V2.git
    ```
2. Download weights from [Depth-Anything-V2-Large](https://huggingface.co/depth-anything/Depth-Anything-V2-Large/resolve/main/depth_anything_v2_vitl.pth?download=true) and place it under `Depth-Anything-V2/checkpoints/`
3. Generate depth maps:
   ```
   python Depth-Anything-V2/run.py --encoder vitl --pred-only --grayscale --img-path <path to input images> --outdir <output path>
   ```
5. Generate a `depth_params.json` file using:
    ```
    python utils/make_depth_scale.py --base_dir <path to colmap> --depths_dir <path to generated depths>
    ```

A new parameter should be set when training if you want to use depth regularization `-d <path to depth maps>`.

## Exposure compensation
To compensate for exposure changes in the different input images we optimize an affine transformation for each image just as in [Hierarchical 3dgs](https://repo-sam.inria.fr/fungraph/hierarchical-3d-gaussians/).  

This can greatly improve reconstruction results for "in the wild" captures, e.g., with a smartphone when the exposure setting of the camera is not fixed. For example results showing the potential benefit and statistics on quality please see here: [Stats for exposure compensation](results.md).

Add the following parameters to enable it:
```
--exposure_lr_init 0.001 --exposure_lr_final 0.0001 --exposure_lr_delay_steps 5000 --exposure_lr_delay_mult 0.001 --train_test_exp
```
Again, other excellent papers have used similar ideas e.g. [NeRF-W](https://nerf-w.github.io/), [URF](https://urban-radiance-fields.github.io/).

## Anti-aliasing
We added the EWA Filter from [Mip Splatting](https://niujinshuchong.github.io/mip-splatting/) in our codebase to remove aliasing. It is disabled by default but you can enable it by adding `--antialiasing` when training on a scene using `train.py` or rendering using `render.py`. Antialiasing can be toggled in the SIBR viewer, it is disabled by default but you should enable it when viewing a scene trained using `--antialiasing`.
![aa](/assets/aa_onoff.gif)
*this scene was trained using `--antialiasing`*.

## SIBR: Top view
> `Views > Top view`

The `Top view` renders the SfM point cloud in another view with the corresponding input cameras and the `Point view` user camera. This allows visualization of how far the viewer is from the input cameras for example.

It is a 3D view so the user can navigate through it just as in the `Point view` (modes available: FPS, trackball, orbit).
<!-- _gif showing the top view, showing it is realtime_ -->
<!-- ![topViewOpen_1.gif](../assets/topViewOpen_1_1709560483017_0.gif) -->
![top view open](assets/top_view_open.gif)

Options are available to customize this view, meshes can be disabled/enabled and their scales can be modified. 
<!-- _gif showing different options_ -->
<!-- ![topViewOptions.gif](../assets/topViewOptions_1709560615266_0.gif) -->
![top view options](assets/top_view_options.gif)
A useful additional functionality is to move to the position of an input image, and progressively fade out to the SfM point view in that position (e.g., to verify camera alignment). Views from input cameras can be displayed in the `Top view` (*note that `--images-path` must be set in the command line*). One can snap the `Top view` camera to the closest input camera from the user camera in the `Point view` by clicking `Top view settings > Cameras > Snap to closest`. 
<!-- _gif showing for a snapped camera the ground truth image with alpha_ -->
<!-- ![topViewImageAlpha.gif](../assets/topViewImageAlpha_1709560852268_0.gif) -->
![top view image alpha](assets/top_view_image_alpha.gif)

## OpenXR support

OpenXR is supported in the branch `gaussian_code_release_openxr`.
Within that branch, you can find documentation for VR support [here](https://gitlab.inria.fr/sibr/sibr_core/-/tree/gaussian_code_release_openxr?ref_type=heads).