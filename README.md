# Ali


[跟我一起写Makefile](https://seisman.github.io/how-to-write-makefile/index.html#)



#!/bin/bash

SHELLDIR=$(dirname "`realpath \"$0\"`")

FORCE_HW_DECODE=true
FORCE_HW_ENCODE=true

echo "WRAPPER OLD: " $0 "$@"

skipnext=false
next_is_input=false
crf=false
codec_is_for_input=true
newargs_prepend=()
newargs_in=()
newargs_out=()
is_input=true
next_is_maxrate=false
for arg in "$@"; do
#  echo $arg
  if $skipnext; then
    skipnext=false
    continue
  fi
  if [ "$arg" = "-crf" ] && $FORCE_HW_ENCODE; then
    skipnext=true
    # `-crf <num>` is not supported. but `-rc vbr/crf` is
    newargs+=("-rc" "crf")
    crf=true
    continue
  fi
  if [ "$arg" = "-x264opts:0" ] && $FORCE_HW_ENCODE; then
    continue
  fi
  if [[ $arg == subme* ]] && $FORCE_HW_ENCODE; then
    continue
  fi
  #if [[ $arg == expr:* ]] && $FORCE_HW_ENCODE; then
  #  continue
  #fi
  if [ "$arg" == "libx264" ] && $FORCE_HW_ENCODE; then arg="h264_nvmpi"; fi
  if [ "$arg" == "libx265" ] && $FORCE_HW_ENCODE; then arg="hevc_nvmpi"; fi
  # Profile mapping
  if [ "$arg" == "veryslow" ] && $FORCE_HW_ENCODE; then arg="slow"; fi
  if [ "$arg" == "slower" ] && $FORCE_HW_ENCODE; then arg="slow"; fi
  if [ "$arg" == "slow" ] && $FORCE_HW_ENCODE; then arg="slow"; fi
  if [ "$arg" == "medium" ] && $FORCE_HW_ENCODE; then arg="medium"; fi
  if [ "$arg" == "fast" ] && $FORCE_HW_ENCODE; then arg="fast"; fi
  if [ "$arg" == "veryfast" ] && $FORCE_HW_ENCODE; then arg="fast"; fi
  if [ "$arg" == "superfast" ] && $FORCE_HW_ENCODE; then arg="ultrafast"; fi
  if [ "$arg" == "ultrafast" ] && $FORCE_HW_ENCODE; then arg="ultrafast"; fi
  if $is_input; then
    newargs_in+=("$arg")
  else
    newargs_out+=("$arg")
  fi
  if $next_is_maxrate && $FORCE_HW_ENCODE; then
    next_is_maxrate=false
    # Set bitrate to maxrate
    # jellyfin only limits bit rate. But the hwencoder needs a suggestion
    newargs_out+=("-b:v")
    newargs_out+=("$arg")
  fi
  if [ "$arg" = "-maxrate" ]; then
    next_is_maxrate=true
  fi

  if $next_is_input; then
    next_is_input=false
    #video_codec=$($SHELLDIR/ffprobe "${newargs_in[@]}" 2>&1 | sed -nE 's/^\s*Stream .* Video: ([[:alnum:]]*).*$/\1/p')
    video_codec=$($SHELLDIR/ffprobe -i "$arg" 2>&1 | sed -nE 's/^\s*Stream .* Video: ([[:alnum:]]*).*$/\1/p')
    #echo "WRAPPER INPUT CODEC $video_codec"
    newargs_prepend=("-c:v" "${video_codec}_nvmpi")
    #newargs_old="${newargs[@]}"
    #newargs=()
    #for arg in "${newargs_prepend[@]}"; do newargs+=("$arg"); done
    #for arg in "${newargs_old[@]}"; do newargs+=("$arg"); done
    #echo "WRAPPER IN: ${newargs[@]}"
    is_input=false
  fi
  if [ "$arg" = "-i" ]; then
    codec_is_for_input=false
    next_is_input=true
  fi

done

if $FORCE_HW_ENCODE && ! $crf; then
  # Default to vbr if crf was not set
  newargs+=("-rc:v" "vbr")
fi

#echo "----"
#for newarg in "${newargs[@]}"; do
#  echo $newarg
#done
#exec $SHELLDIR/showargs.py "${newargs[@]}"

echo "WRAPPER NEW: " $SHELLDIR/ffmpeg "${newargs_prepend[@]}" "${newargs_in[@]}" "${newargs_out[@]}"

exec $SHELLDIR/ffmpeg "${newargs_prepend[@]}" "${newargs_in[@]}" "${newargs_out[@]}"
#echo "WRAPPER OUT " $($SHELLDIR/ffmpeg "${newargs_prepend[@]}" "${newargs_in[@]}" "${newargs_out[@]}" 2>&1)
