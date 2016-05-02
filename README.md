# Oyster
©2016 IFeelBloated, Oyster Python Module for VapourSynth
## License
LGPL v2.1
## Description
Oyster is a top class de-noising filter against compression artifacts in terms of quality, it is outrageously slow and deadly expensive at computational costs.

Each pixel will be restored by the weighted average of its neighbors, weights generated by Block-Matching and Pixel-Matching algorithms, spatially and temporally.

It is designed for photographic videos, but works on CGI like cartoons and anime also.
## Requirements
- NNEDI3
- KNLMeansCL
- BM3D
- FMTConv
- MVTools (floating point ver)

## Function List
- Basic
- Final

## Formats
- Bitdepth: 32bits floating point
- Color Space: Gray, RGB, YUV 4:4:4 (subsampled YUV formats are not supported)
- Scan Type: Progressive

## Notes
- DO NOT upsample your video to YUV 4:4:4 or RGB before processing if it's not natively full-sampled, just pass the luminance plane as a gray clip and merge the processed luma with the source chroma, fake 4:4:4 is toxic as the low-res chroma will jeopardize the correctness of weight calculation (especially on Pixel-Matching), and then the quality degradation on luma sets in.
- DO NOT crop your video before processing, it will destroy the macroblock boundary detecting.
- You might wanna try waifu2x instead if your video is of CGI-like content, Oyster is times slower than waifu2x and designed specifically for photographic videos.

## Details
### Basic
The basic estimating features 3 stages:

- Cleansing<br />
  an NLMeans filtering with aggressive parameters will be applied to wipe all the artifacts away
- Motion Compensation<br />
  subpixel Block-Matching based motion compensation will be applied to recover significant structure loss.
- Refining (optional) <br />
  Pixel-Matching looped refining will be performed to recover fine and delicate details, disabled at level=2

it serves as a reference to the later final estimating
```python
Basic (src, level=1, \
       radius=6, h=6.4, pel=4, pel_precise=True, thscd1=10000, thscd2=255, \
       deblock=True, deblock_thr=0.03125, deblock_elast=0.015625, \
       lowpass=8)
```
- src<br />
  clip to be processed
- level<br />
  could be 1 or 2, default 1, de-noise level, level1 works on typical compression artifacts, level2 works on severe compression artifacts
- radius<br />
  temporal radius, 2*radius+1 frames will be referenced and the current frame will be the middle
- h<br />
  filtering strength
- pel, thscd1, thscd2<br />
  read the MVTools doc
- pel_precise<br />
  sub-pixel interpolation, True = NNEDI(Neural Network Edge Directed Interpolation), False = wiener interpolation
- deblock, deblock_thr, deblock_elast<br />
  - deblock: deblocking switch for level1, the macroblock boundaries will be replaced with the blend of level1 and level2 when set True, the parameter does not work at level2
  - deblock_thr: threshold of the pixel difference between level1 and level2, if the absolute difference > deblock_thr, take the pixel from level2, else take the pixel from level1, ranges from 0.0 to 1.0
  - deblock_elast: elasticity of deblock_thr, ranges from 0.0 to deblock_thr
- lowpass<br />
  compression artifacts are high frequency artifacts, lowpass is the frequency threshold, frequencies below it will not be filtered, ranges from 1 to 100

### Final
The final estimating features 4 stages:

- Cleansing<br />
  wild motion compensated block averaging trying to wipe intense artifacts away while maintaining significant structures standing still
- Refining (optional) <br />
  Pixel-Matching looped refining will be performed to recover fine and delicate details, disabled at level=2
- Further Cleaning<br />
  less wild BM3D trying to sweep residual subtle artifacts away
- Further Refining<br />
  another round of Pixel-Matching refining to recover delicate details lost in the BM3D filtering

a reference clip is required for the final estimating
```python
Final (src, ref, level=1, \
       radius=6, h=6.4, sigma=12.0, pel=4, pel_precise=True, thsad=2000, thscd1=10000, thscd2=255, \
       block_size=8, block_step=1, group_size=32, bm_range=24, bm_step=1, ps_num=2, ps_range=8, ps_step=1, \
       deblock=True, deblock_thr=0.03125, deblock_elast=0.015625, \
       lowpass=8):
```
- ref<br />
  reference clip generated by basic estimating
- thsad<br />
  read the MVTools doc, DO NOT set it under 1200, for the record
- sigma, block_size, block_step, group_size, bm_range, bm_step, ps_num, ps_range, ps_step<br />
  read the BM3D doc

## Demos
```python
import vapoursynth as vs
import sys
if "Oyster" in sys.modules: del sys.modules["Oyster"]
import Oyster
core = vs.get_core()

clp = #typical yv12 stuff
clp = core.fmtc.bitdepth(clp,bits=32,fulls=False,fulld=True)
y = core.std.ShufflePlanes(clp,0,vs.GRAY)

ref = Oyster.Basic(y)
y = Oyster.Final(y,ref)

clp =  core.std.ShufflePlanes([y,clp], [0,1,2], vs.YUV)
clp.set_output()
```
