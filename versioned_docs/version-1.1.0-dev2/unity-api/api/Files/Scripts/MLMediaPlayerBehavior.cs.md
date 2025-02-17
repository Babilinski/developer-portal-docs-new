---

title: MLMediaPlayerBehavior.cs

---


# MLMediaPlayerBehavior.cs









## Source code

```csharp
// %BANNER_BEGIN%
// ---------------------------------------------------------------------
// %COPYRIGHT_BEGIN%
// Copyright (c) (2021-2022) Magic Leap, Inc. All Rights Reserved.
// Use of this file is governed by the Software License Agreement, located here: https://www.magicleap.com/software-license-agreement-ml2
// Terms and conditions applicable to third-party materials accompanying this distribution may also be found in the top-level NOTICE file appearing herein.
// %COPYRIGHT_END%
// ---------------------------------------------------------------------
// %BANNER_END%

using System;
using System.IO;
using UnityEngine;
using UnityEngine.Video;
using UnityEngine.XR.MagicLeap;

namespace MagicLeap.Core
{
    public class MLMediaPlayerBehavior : MonoBehaviour
    {
        public enum PathSourceType
        {
            Web,
            StreamingAssets,
            LocalPath,
        }

        [SerializeField, Tooltip("MeshRenderer to display media")]
        private Renderer screen = null;

        [SerializeField, Tooltip("A reference of media player's renderer's material.")]
        private Material videoRenderMaterial = null;

        [SerializeField, Tooltip("Used to indicate the stereo mode set for the media player.")]
        private Video3DLayout stereoMode = Video3DLayout.No3D;

        [SerializeField, Tooltip("Used to set media player content loop.")]
        private bool isLooping;

        [Tooltip("Used to indicate the source path for the media player.")]
        public PathSourceType pathSourceType;

        [Tooltip("URI/Path of the media to be played")]
        public string source;

        [SerializeField, Tooltip("A reference of the media player texture.")]
        private RenderTexture mediaPlayerTexture = null;

        public bool IsBuffering { get; private set; } = false;
        public bool IsSeeking { get; private set; } = false;
        public long DurationInMiliseconds { get; private set; } = 0;

        public event Action OnPrepared;
        public event Action OnPlay;
        public event Action OnPause;
        public event Action OnStop;
        public event Action OnReset;
        public event Action OnCompletion;
        public event Action<float> OnBufferingUpdate;
        public event Action<MLMedia.Player.Info> OnInfo;
        public event Action OnSeekComplete;
        public event Action<MLMedia.Player.Track> OnTrackSelected;
        public event Action<string> OnCaptionsText;
        public event Action<float> OnUpdateTimeline;
        public event Action<long> OnUpdateElapsedTime;
        public event Action<bool> OnIsBufferingChanged;
        public event Action<MLNativeSurfaceYcbcrRenderer> OnVideoRendererInitialized;

        private long currentPositionInMiliseconds = 0;
        private bool hasSetSourceURI = false;
        private int videoWidth, videoHeight;

        public MLMedia.Player MediaPlayer
        {
            get
            {
                if (_mediaPlayer == null)
                {
                    var mediaPlayer = new MLMedia.Player(out MLResult result);

                    if (!result.IsOk)
                        throw new Exception($"Media Player initialization result with error {result}.");

                    _mediaPlayer = mediaPlayer;

                    _mediaPlayer.OnPrepared += HandleOnPrepare;
                    _mediaPlayer.OnVideoSizeChanged += HandleOnVideoSizeChanged;
                    _mediaPlayer.OnPlay += HandleOnPlay;
                    _mediaPlayer.OnPause += HandleOnPause;
                    _mediaPlayer.OnStop += HandleOnStop;
                    _mediaPlayer.OnCompletion += HandleOnCompletion;
                    _mediaPlayer.OnBufferingUpdate += HandleOnBufferingUpdate;
                    _mediaPlayer.OnInfo += HandleOnInfo;
                    _mediaPlayer.OnSeekComplete += HandleOnSeekComplete;
                    _mediaPlayer.OnCEA608 += HandleOnCaptionsText;
                    _mediaPlayer.OnCEA708 += HandleOnCaptionsText;
                    _mediaPlayer.OnTimedText += HandleOnCaptionsText;
                    _mediaPlayer.OnTrackSelected += HandleOnTrackSelected;
                    _mediaPlayer.OnResetComplete += HandleOnResetComplete;
                }

                return _mediaPlayer;
            }
        }

        public bool IsPlaying => _mediaPlayer is { IsPlaying: true };
        public bool IsPrepared => _mediaPlayer is { IsPrepared: true };

        private MLMedia.Player _mediaPlayer;

        void Update()
        {
            if (!Application.isEditor && MediaPlayer.IsPlaying && MediaPlayer.VideoRenderer != null)
            {
                MediaPlayer.VideoRenderer.Render();
            }
            if (DurationInMiliseconds > 0 && IsPlaying && !IsSeeking && !IsBuffering)
            {
                UpdateTimeline();
            }
        }

        private void OnDestroy()
        {
            if (_mediaPlayer != null)
            {
                _mediaPlayer.OnPrepared -= HandleOnPrepare;
                _mediaPlayer.OnVideoSizeChanged -= HandleOnVideoSizeChanged;
                _mediaPlayer.OnPlay -= HandleOnPlay;
                _mediaPlayer.OnPause -= HandleOnPause;
                _mediaPlayer.OnStop -= HandleOnStop;
                _mediaPlayer.OnCompletion -= HandleOnCompletion;
                _mediaPlayer.OnBufferingUpdate -= HandleOnBufferingUpdate;
                _mediaPlayer.OnInfo -= HandleOnInfo;
                _mediaPlayer.OnSeekComplete -= HandleOnSeekComplete;
                _mediaPlayer.OnCEA608 -= HandleOnCaptionsText;
                _mediaPlayer.OnCEA708 -= HandleOnCaptionsText;
                _mediaPlayer.OnTimedText -= HandleOnCaptionsText;
                _mediaPlayer.OnTrackSelected -= HandleOnTrackSelected;
                _mediaPlayer.OnResetComplete -= HandleOnResetComplete;

                StopMLMediaPlayer();
                _mediaPlayer.Reset();
                _mediaPlayer.Destroy();
            }
        }

        public void PrepareMLMediaPlayer()
        {
            if (MediaPlayer.IsPrepared)
            {
                MLPluginLog.Warning("Media Player is already prepared.");
                return;
            }

            if (screen == null)
            {
                MLPluginLog.Warning("PrepareMLMediaPlayer failed, no valid screen found");
                return;
            }

            // Create a Url with provided string and test if its a local file
            MLResult result = MLResult.Create(MLResult.Code.UnspecifiedFailure);
            MLMedia.Player mlPlayer = MediaPlayer as MLMedia.Player;

            if (pathSourceType == PathSourceType.Web)
            {
                if (!MLResult.DidNativeCallSucceed(MLPermissions.CheckPermission(MLPermission.Internet).Result, nameof(MLPermissions.CheckPermission)))
                {
                    Debug.LogError($"Using PathSourceType.Web requires {MLPermission.Internet} permission included in AndroidManifest.xml");
                    result = MLResult.Create(MLResult.Code.PermissionDenied);
                }
                else
                {
                    if (!hasSetSourceURI)
                    {
                        result = mlPlayer.SetSourceURI(source);
                        hasSetSourceURI = true;
                    }
                }
            }
            else if (pathSourceType == PathSourceType.StreamingAssets)
            {
                string streamingPath = Path.Combine(Application.streamingAssetsPath, source);
                result = mlPlayer.SetStreamingSourcePath(streamingPath);
            }
            else if (pathSourceType is PathSourceType.LocalPath)
            {
                string persistentDataPath = Path.Combine(Application.persistentDataPath, source);
                result = mlPlayer.SetSourcePath(persistentDataPath);
            }

            if (!result.IsOk)
            {
                string message = "PrepareMLMediaPlayer failed, source could not be set.";
                MLPluginLog.Warning(message);
                return;
            }

            mlPlayer.PreparePlayerAsync();
        }

        public void StopMLMediaPlayer()
        {
            if (_mediaPlayer == null)
                return;

            if (_mediaPlayer.IsPrepared)
            {
                _mediaPlayer.Stop();
            }
        }

        private void SetRendererTexture(RenderTexture texture)
        {
            if (mediaPlayerTexture != texture)
            {
                Destroy(mediaPlayerTexture);
                mediaPlayerTexture = null;
            }

            if (mediaPlayerTexture == null)
            {
                // Create texture with given dimensions
                mediaPlayerTexture = texture;
            }

            // Set texture on quad
            screen.material.SetTexture("_MainTex", this.mediaPlayerTexture);

            MediaPlayer.CreateVideoRenderer((uint)texture.width, (uint)texture.height);
            MediaPlayer.VideoRenderer.SetRenderBuffer(this.mediaPlayerTexture);
        }

        private void InitializeMLMediaPlayerRenders(int width, int height)
        {
            if (videoRenderMaterial == null)
            {
                MLPluginLog.Warning($"MLMediaPlayerBehavior failed to initialize, video render material is missing.");
                return;
            }

            TryApplyVideoRenderMaterial(videoRenderMaterial);
            TryApplyStereoMode();

            if (mediaPlayerTexture == null)
                CreateTexture(width, height);
            else
                SetRendererTexture(mediaPlayerTexture);

            float aspectRatio = width / (float)height;
            transform.localScale = new Vector3(transform.localScale.y * aspectRatio, transform.localScale.y, 1);

            OnVideoRendererInitialized?.Invoke(MediaPlayer.VideoRenderer);
        }

        private void TryApplyStereoMode()
        {
            if (this.videoRenderMaterial != null)
            {
                if (this.videoRenderMaterial.HasProperty("_VideoStereoMode"))
                {
                    this.videoRenderMaterial.SetInt("_VideoStereoMode", (int)this.stereoMode);
                }
                else if (this.stereoMode == Video3DLayout.No3D)
                {
                    MLPluginLog.Warning(
                        $"MLMediaPlayerBehavior failed to apply {stereoMode} StereoMode, material is missing \"_VideoStereoMode\" property");
                }
            }
        }

        private void TryApplyVideoRenderMaterial(Material rendererMaterial)
        {
            if (this.screen != null)
            {
                this.screen.material = rendererMaterial;

                // Accessing the renderer's material automatically instantiates it and makes it unique to this renderer, so keep a reference.
                this.videoRenderMaterial = this.screen.material;
            }
        }

        private void CreateTexture(int width, int height)
        {
            width = Mathf.Max(width, 1);
            height = Mathf.Max(height, 1);

            if (mediaPlayerTexture != null && (mediaPlayerTexture.width != width || mediaPlayerTexture.height != height))
            {
                Destroy(mediaPlayerTexture);
                mediaPlayerTexture = null;
            }

            if (mediaPlayerTexture == null)
            {
                // Create texture with given dimensions
                mediaPlayerTexture = new RenderTexture(width, height, 0, RenderTextureFormat.ARGB32);

                // Set texture on quad
                screen.material.SetTexture("_MainTex", this.mediaPlayerTexture);
            }

            MediaPlayer.CreateVideoRenderer((uint)width, (uint)height);
            MediaPlayer.VideoRenderer.SetRenderBuffer(this.mediaPlayerTexture);
        }

        public void Pause()
        {
            if (!MediaPlayer.IsPrepared)
                MediaPlayer.OnPrepared -= PlayOnPrepared;

            if (MediaPlayer.IsPlaying)
                MediaPlayer.Pause();
        }

        public void Play()
        {
            if (!MediaPlayer.IsPrepared)
            {
                MediaPlayer.OnPrepared += PlayOnPrepared;
                PrepareMLMediaPlayer();
            }
            else
                MediaPlayer.Resume();
        }

        private void PlayOnPrepared(MLMedia.Player mediaplayer)
        {
            MediaPlayer.OnPrepared -= PlayOnPrepared;

            MediaPlayer.SetLooping(isLooping);
            MediaPlayer.Play();
        }

        public void Seek(float ms)
        {
            IsSeeking = true;

            // Jump backwards or forwards in the video by 'time'
            float position = Mathf.Clamp(currentPositionInMiliseconds + ms, 0, DurationInMiliseconds);
            float currentPositionRatio = position / DurationInMiliseconds;
            int lastTimeSoughtMs = Mathf.RoundToInt((currentPositionRatio * DurationInMiliseconds));
            OnUpdateTimeline?.Invoke(currentPositionRatio);
            OnUpdateElapsedTime?.Invoke(lastTimeSoughtMs);
            MediaPlayer.Seek((int)position, MLMedia.Player.SeekMode.Closest);
        }

        public void SeekTo(float ms)
        {
            IsSeeking = true;

            float currentPositionRatio = (float)ms / DurationInMiliseconds;
            int lastTimeSoughtMs = Mathf.RoundToInt(currentPositionRatio * DurationInMiliseconds);
            OnUpdateTimeline?.Invoke(currentPositionRatio);
            OnUpdateElapsedTime?.Invoke(lastTimeSoughtMs);
            MediaPlayer.Seek((int)ms, MLMedia.Player.SeekMode.ClosestSync);
        }

        private void UpdateTimeline()
        {
            // Only poll the position once per frame to prevent seeking by miniscule amounts.
            currentPositionInMiliseconds = MediaPlayer.GetPositionMilliseconds();
            float currentPositionRatio = (float)currentPositionInMiliseconds / DurationInMiliseconds;
            OnUpdateTimeline?.Invoke(currentPositionRatio);
            OnUpdateElapsedTime?.Invoke(currentPositionInMiliseconds);
        }

        private void HandleOnPrepare(MLMedia.Player mediaplayer)
        {
            DurationInMiliseconds = MediaPlayer.GetDurationMilliseconds();
            hasSetSourceURI = false;
            OnPrepared?.Invoke();
        }

        private void HandleOnVideoSizeChanged(MLMedia.Player player, int width, int height)
        {
            if (width == videoWidth && height == videoHeight)
                return;

            videoWidth = width;
            videoHeight = height;
            InitializeMLMediaPlayerRenders(width, height);
        }

        private void HandleOnPlay(MLMedia.Player mediaPlayer)
        {
            OnPlay?.Invoke();
        }

        private void HandleOnPause(MLMedia.Player mediaPlayer)
        {
            OnPause?.Invoke();
        }

        private void HandleOnStop(MLMedia.Player mediaplayer)
        {
            OnStop?.Invoke();
        }

        private void HandleOnCompletion(MLMedia.Player mediaPlayer)
        {
            OnCompletion?.Invoke();
        }

        private void HandleOnResetComplete(MLMedia.Player mediaPlayer)
        {
            PrepareMLMediaPlayer();
            OnReset?.Invoke();
        }

        private void HandleOnBufferingUpdate(MLMedia.Player mediaPlayer, float percent)
        {
            OnBufferingUpdate?.Invoke(percent);
        }

        private void HandleOnInfo(MLMedia.Player mediaPlayer, MLMedia.Player.Info info)
        {
            switch (info)
            {
                case MLMedia.Player.Info.NetworkBandwidth:
                    // source media is not local
                    // the parameter extra would contain bandwidth in kbps
                    break;
                case MLMedia.Player.Info.BufferingStart:
                    IsBuffering = true;
                    OnIsBufferingChanged?.Invoke(true);
                    break;
                case MLMedia.Player.Info.BufferingEnd:
                    IsBuffering = false;
                    OnIsBufferingChanged?.Invoke(false);
                    break;
            }
        }

        private void HandleOnSeekComplete(MLMedia.Player mediaPlayer)
        {
            IsSeeking = false;
            OnSeekComplete?.Invoke();
        }

        private void HandleOnTrackSelected(MLMedia.Player mediaPlayer, MLMedia.Player.Track track)
        {
            OnTrackSelected?.Invoke(track);
        }

        private void HandleOnCaptionsText(MLMedia.Player mediaPlayer, string text)
        {
            OnCaptionsText?.Invoke(text);
        }
    }
}
```



