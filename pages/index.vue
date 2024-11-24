<template>
  <iframe src="https://cursorful.com/welcome" scrolling="no"></iframe>
</template>

<script setup lang="ts">
;(async () => {
  // very bizarre, with DRAW_TO_CANVAS = true, MediaRecorder recording displays colors correctly
  const DRAW_TO_CANVAS = false

  const frameRate = 60
  const bitRate = 35_000_000
  const mediaStream = await navigator.mediaDevices.getDisplayMedia({
    video: {
      frameRate,
      displaySurface: 'monitor',
    },
  })

  const videoTrack = mediaStream.getVideoTracks()[0]

  const videoTrackSettings = videoTrack.getSettings()

  const canvas = document.createElement('canvas')
  canvas.width = videoTrackSettings.width!
  canvas.height = videoTrackSettings.height!
  const ctx = canvas.getContext('2d', {
    // ctx alpha needs to be set to false OR the VideoFrame alpha set to 'discard' for the MediaRecorder recording to display colors correctly
    alpha: false,
  })!

  const trackProcessor = new MediaStreamTrackProcessor({
    track: videoTrack,
  })
  const trackGenerator = new MediaStreamTrackGenerator({
    kind: 'video',
  })

  let f = 0
  trackProcessor.readable
    .pipeThrough(
      new TransformStream<VideoFrame, VideoFrame>({
        transform(frame, controller) {
          if (DRAW_TO_CANVAS) {
            ctx.drawImage(frame, 0, 0)
            frame.close()
            frame = new VideoFrame(canvas, {
              timestamp: frame.timestamp,
              // alpha: 'discard' OR ctx alpha set to false are necessary for the MediaRecorder recording to display colors correctly
              alpha: 'discard',
            })
          }

          controller.enqueue(frame)
          f++
        },
      })
    )
    .pipeTo(trackGenerator.writable)

  const transformedMediaStream = new MediaStream([trackGenerator])

  // compare with MediaRecorder
  const mediaRecorder = new MediaRecorder(transformedMediaStream, {
    mimeType: 'video/mp4',
    videoBitsPerSecond: bitRate,
  })
  const chunks: Blob[] = []
  mediaRecorder.ondataavailable = (e) => {
    chunks.push(e.data)
  }
  mediaRecorder.start()

  mediaRecorder.onstop = () => {
    window.open(
      URL.createObjectURL(
        new Blob(chunks, {
          type: 'video/mp4',
        })
      ),
      '_blank'
    )
  }
})()
</script>

<style>
body {
  margin: 0;
}

iframe {
  width: 100vw;
  height: 100vh;
}
</style>
