<script>
  import { getContext, onMount, onDestroy } from "svelte"
  import "./videojs.css"
  import videojs from "video.js"
  import { WebVTTParser } from 'webvtt-parser';

  export let videoUrl
  export let thumbnailPath
  export let controlsEnabled = true
  export let videoTitle = ""
  export let chaptersData
  export let subtitle
  export let onTimeUpdate
  export let preload = false
  // export let tracks = []

  const { styleable, builderStore } = getContext("sdk")
  const component = getContext("component")

  let videoEle
  let ready = false
  let domLoaded = false
  let player
  let playing = false

  $: redrawOptions = {
    videoUrl,
    thumbnailPath,
    chaptersData,
    subtitle
  }

  $: if(redrawOptions && ready && $builderStore?.inBuilder){
    console.log("should dispose and rebuild.")
  }

  const trackBlob = (sourceText, kind) => {
    const blob = new Blob([sourceText], {
      type: "text/vtt",
    });
    const blobURL = URL.createObjectURL(blob)
    return { kind:kind, src:blobURL , srclang:"en", label:"English" }
  }

  $: if(!ready && domLoaded) {
    ready = true
    
    let tracks = []
    if(chaptersData){
      const parser = new WebVTTParser();
      const tree = parser.parse(chaptersData, 'chapters');
      console.log(tree)
      if(!tree.errors.length){
        tracks.push(trackBlob(chaptersData, "chapters"))
      }
    }
    if(subtitle){
      const parser = new WebVTTParser();
      const tree = parser.parse(chaptersData, 'subtitles');
      if(!tree.errors.length){
        tracks.push(trackBlob(subtitle, "subtitles"))
      }
    }

    player = videojs(videoEle, { 
      controls : controlsEnabled,
      tracks,
      fluid: true,
      preload: preload ? "auto" : null
    });

    player.on('play', () => { 
      playing = true 
    })
    player.on('pause', () => { 
      playing = false 
    })
    player.on('timeupdate', () => {
      updateTime(player.currentTime())
    })
  }

  const updateTime = async (time) => {
    if (!onTimeUpdate) {
      return
    }
    console.log("hello?")
    const res = await onTimeUpdate({ videoSeconds : time })
    console.log("timeupdated ", res)
  }

  onMount(() => {
    domLoaded = true
  })

  onDestroy(() => {
    console.log("DISPOSE")
    player.dispose();
  })
</script>

<div use:styleable={$component.styles}>
  {#if videoUrl}
    {#key redrawOptions}
      <div class="video-wrap">
        <video bind:this={videoEle} class="video-js vjs-default-skin" data-setup={"{}"} poster={thumbnailPath || null}>
          <source src={videoUrl} type="video/mp4">
          <track kind="captions">
        </video>
        {#if !playing}
          <div class="video-overlay">{videoTitle}</div>
        {/if}
      </div>
    {/key}
  {:else}
    Default experience.
  {/if}
</div>

<style>
  .video-wrap { 
    position: relative
  }
  .video-wrap,
  .video-wrap > *,
  video
  {
    max-width: 100%!important;
  }
  .video-overlay {
    color: white;
    padding: 16px;
    font-size: 16px;
    position: absolute;
    top: 0px;
    left: 0px;
    width: 100%;
    height: 100%;
    pointer-events: none;
    background: linear-gradient(to bottom, rgba(0, 0, 0, .6) 0%, rgba(0, 0, 0, .3) 30%, rgba(0, 0, 0, 0) 100%);
  }
</style>
