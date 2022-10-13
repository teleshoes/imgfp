# imgfp
    Copyright © 2022 by Elliot Wolk.
    License: GPLv3
    based on: `findimagedupes`
        Copyright © 2006-2022 by Jonathan H N Chin <code@jhnc.org>.

imgfp creates 'fingerprints' (perceptual hashes) of visual images.
It is intended to be highly configurable, script-friendly and command-line friendly.

Main differences compared with `findimagedupes`:
  - imgfp has cmdline options to configure fingerprint (supports rgb/grayscale fingerprints)
  - imgfp has cmdline options to print yes/no matching
  - imgfp caches fingerprints by default, in ~/.cache/imgfp/
  - imgfp checks mtime + filesize in cache
  - imgfp is more script-friendly (dupe img output is 1 file per line, with an index)
  - imgfp fingerprints are pixel data in hex (instead of base64-encoded mono format)
  - imgfp has slightly improved accuracy (at least, for all my use cases, YMMV)
  - imgfp has slower fingerprint-comparison than findimagedupes
  - imgfp assumes GNU/Linux directory structures

## Dependencies:
    perl
    Graphics::Magick  (libgraphics-magick-perl)

## Usage:
```
Usage:
  imgfp -h | --help
    show this message

  imgfp [OPTS] --compare IMG_SRC IMG_COMP [IMG_COMP ..]
  imgfp [OPTS] --diff    IMG_SRC IMG_COMP [IMG_COMP ..]
  imgfp [OPTS] -d        IMG_SRC IMG_COMP [IMG_COMP ..]
    find the SIMILARITY_PCT between the fingerprints of IMG_SRC and IMG_COMP
      -calculate the fingerprints of IMG_SRC and each IMG_COMP as in --fp
      -sum the absolute difference of each channel (RGB or gray) of each pixel
      -subtract this value from the maximum possible difference
      -express as a percentage

  imgfp [OPTS] --match   IMG_SRC IMG_COMP [IMG_COMP ..]
  imgfp [OPTS] --matches IMG_SRC IMG_COMP [IMG_COMP ..]
  imgfp [OPTS] -m        IMG_SRC IMG_COMP [IMG_COMP ..]
    for each IMG_COMP, get the SIMILARITY_PCT with IMG_SRC as in --compare,
      and print 'yes' or 'no' if SIMILARITY_PCT >= THRESHOLD_PCT
    default threshold is 95%

  imgfp [OPTS] --find-dupes IMG IMG [IMG ..]
    print matching IMG filesets as in --match,
      formatted as one IMG file per line, prefixed with a matching group index
    -for each IMG:
      -use IMG as IMG_SRC
      -assign MATCH_GROUP_INDEX based on IMG_SRC position in file list, starting with 1
      -use all remaining IMG args (not used as IMG_SRC yet) as IMG_COMP
      -perform --match with IMG_SRC and each IMG_COMP
      -if matches are found for at least one IMG_COMP, print:
         MATCH_GROUP_INDEX:IMG_SRC
         MATCH_GROUP_INDEX:IMG_COMP
         MATCH_GROUP_INDEX:IMG_COMP
         ...
      -otherwise, print nothing
    e.g.: --find-dupes a1.jpg a2.jpg b1.jpg c1.jpg a3.jpg b2.jpg d1.jpg
            1:a1.jpg
            1:a2.jpg
            1:a3.jpg
            3:b1.jpg
            3:b2.jpg

  imgfp [OPTS] --find-dupes-oneline IMG IMG [IMG ..]
    same as --find-dupes, except print each matching group on a single line,
    with a space character separating each IMG, and not MATCH_GROUP_INDEX
    e.g.: --find-image-dupes a1.jpg a2.jpg b1.jpg c1.jpg a3.jpg b2.jpg d1.jpg
            a1.jpg a2.jpg a3.jpg
            b1.jpg b2.jpg

  imgfp [OPTS] --fingerprint IMG [IMG ..]
  imgfp [OPTS] --fp          IMG [IMG ..]
  imgfp [OPTS] IMG [IMG ..]
    calculate a 'fingerprint' image of IMG, and print the pixels as hex chars
      -(A) load image
      -(B) transform image
        -(B0) if --quick, perform optional initial resize,
              to improve performance of other image transforms
        -(B1) blur - blur the image with a radius of 1.5% the maximum dimension
              (blur is ~80% of the fingerprint generation time)
        -(B2) normalize - modestly increase contrast by stretching intensity range
                -find dark intensity that 2% of pixels are darker than
                -find light intensity that 1% of pixels are whiter than
                -linearly scale intensity of all pixels between these two values
        -(B3) equalize - maximize contrast by flattening the intensity histogram
                -ensure the same number of pixels exist for each gray level
        -(B4) adjust aspect ratio - optionallty add a border to fit into a square
                -if set, ensures that stretching the image affects the fingerprint
        -(B5) resize - resize image to a (small) square with pixel sampling
        -(B6) quantize - reduce colorspace to 12-bit-rgb/4-bit-gray/1-bit-mono
                -for --mono, first threshold the pixels at 50% intensity range
                -then quantize using the ImageMagick quantize algorithm
      -(C) debug output - optionally write final image to a bmp
      -(D) extract pixel data as single bitstring (left-to-right, top-to-bottom)
      -(E) convert to hex

  imgfp [OPTS] --test DIR [DIR ..]
    compare groups of known-similar images, organized in directories
    -for each DIR:
      -compare each IMG file in same DIR to each other as in --compare
        -take the minimum value as MIN_MATCH_PCT
      -for each other dir OTHER_DIR
        -compare each IMG file in DIR to IMG files in OTHER_DIR as in --compare
        -take the maximum value for each dir as MAX_MISMATCH_PCT
      -print DIR: MIN_MATCH_PCT | MAX_MISMATCH_PCT MAX_MISMATCH_PCT ..

  IMAGE_FILE = IMG_SRC | IMG_COMP | IMG
    path to a visual image file
    if '--' is given, read files, one per line, from STDIN

  OPTS
    PRESETS
      --good
        same as: --rgb --size=32 --quick
          PROS: very accurate, moderately fast to generate fingerprints
          CONS: large cache disk space, slow to compare to many files
      --slow
        same as: --rgb --size=64
          PROS: extremely accurate
          CONS: very slow, large cache disk space
      --fast
        same as: --mono --size=16 --quick --threshold=90
          PROS: very fast, very small cache disk space
          CONS: moderate accuracy

    FINGERPRINT SIZE
      Control the height and width of the fingerprint image square.
      The length, in characters, of the hex fingerprint is:
        SQUARE_SIZE^2 * COLORSPACE_BITS / 4
      The default is --size=16

      -s SQUARE_SIZE | --size SQUARE_SIZE | --square-size SQUARE_SIZE
      --size=SQUARE_SIZE | --square-size=SQUARE_SIZE
        height and width in pixels of the square fingerprint image
        (odd SQUARE_SIZE results in padding for --mono fingerprints)

    FINGERPRINT COLORSPACE
      Control the number of bits per pixel in the fingerprint.
      The default is mono1

      --rgb | --color
        same as: --colorspace=rgb12
      --gray | --grayscale
        same as: --colorspace=gray4
      --mono | --bw
        same as: --colorspace=mono1

      --colorspace COLORSPACE
      --colorspace=COLORSPACE
        COLORSPACE
          rgb12  = 12-bit RGB  (COLORSPACE_BITS=12)
                     every 1 pixel is 3 hex chars
                     e.g.: white black white black = 'fff000fff000'
          gray4  = 4-bit grayscale (COLORSPACE_BITS=4)
                     every 1 pixel is 1 hex char
                     e.g.: white black white black = 'f0f0'
          mono1  = 1-bit mono (COLORSPACE_BITS=1)
                     every 4 pixels is 1 hex char
                     e.g.: white black white black = 'a'

    FINGERPRINT ASPECT RATIO
      Control whether a changed aspect ratio affects the fingerprint.
      The default is SQUARE_MODE=pad

      --stretch | --ignore-aspect-ratio
        same as: --square-mode=stretch

      --pad | --keep-aspect-ratio
        same as: --square-mode=pad

      --square-mode=SQUARE_MODE
        SQUARE_MODE=pad
          add a top/bottom or left/right black border to the fingerprint to fit in a square
          this ensures that changing the aspect ratio of an image has a LARGE impact on fingerprint

        SQUARE_MODE=stretch
          stretch the fingerprint (with pixel sampling) to fit in a square
          this ensures that changing the aspect ratio of an image has SMALL impact on fingerprint

    PERFORMANCE
      Control whether to initially resize the image before performing transformations.
      Blur in particular is very slow on larger images.
      The default is QUICK_RESIZE_PX=0 (--no-quick)

      --quick
        same as: --quick-resize=128

      --no-quick
        same as: --quick-resize=0

      --quick-resize=QUICK_RESIZE_PX
      --quick-resize QUICK_RESIZE_PX
        drastically increase performance with a moderate drop in accuracy,
          by resizing image to <QUICK_RESIZE_PX>x<QUICK_RESIZE_PX> before blurring
        smaller QUICK_RESIZE_PX leads to better performance and worse accuracy
        if QUICK_RESIZE_PX is 0, do not perform resize
        include in cache filename:
          QUICK_RESIZE_FMT = no-quick | quick<QUICK_RESIZE_PX>

          e.g.:
            --quick-resize=128 is very fast, and still somewhat accurate
            --quick-resize=1024 is fast, and fairly accurate
            --quick-resize=2048 does not improve performance, except for very large input images

    FINGERPRINT COMPARISON
      Control the threshold at which --match prints yes.
      The default is --threshold=95%

      -t THRESHOLD_PCT | --threshold THRESHOLD_PCT
      --threshold=THRESHOLD_PCT
        fingerprints with a SIMILARITY_PCT >= THRESHOLD_PCT print 'yes' in --match
        e.g.: -t 90.0   --threshold=90%   --threshold 86.714

    FINGERPRINT CACHE
      Cache fingerprints in CACHE_FILE to improve performance of future calls.
      Files must match filesize, mtime, and absolute path, or they will be recalculated.
      The default cache mode is rw
      The default CACHE_FILE changes with fingerprint params:
          /home/wolke/.cache/imgfp/<COLORSPACE>_<SQUARE_SIZE>px_<SQUARE_MODE>_<QUICK_RESIZE_FMT>.txt

      --cache-file CACHE_FILE
      --cache-file=CACHE_FILE
        override CACHE_FILE for reading and writing
        NOTE: Overriding the CACHE_FILE *ignores* fingerprint control options.
              Do not use the same CACHE_FILE with different cmdline options,
                or the results will make no sense.
              (The default CACHE_FILE automatically changes based on fingerprint params.)

      --no-cache
        do not read or write CACHE_FILE
        (always calculate all fingerprints)

      --ro | --read-only-cache
        allow read-only access to CACHE_FILE
        (skip existing fingerprints, but do not cache newly calculated fingerprints)

      --rw | --read-write-cache
        allow read/write access to CACHE_FILE
        (skip existing fingerprints, AND write newly calculated fingerprints)

    MISC
      --debug-dir DEBUG_DIR
      --debug-dir=DEBUG_DIR
        write fingerprint images in BMP format to DEBUG_DIR
```
