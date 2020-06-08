# 请求 mpd 及播放
MediaPlayer.initialize -> MediaPlayer.initializePlayback -> StreamController.load -> ManifestLoader.load -> XlinkController.resolveManifestOnLoad -> Events.ORIGINAL_MANIFEST_LOADED

中间步骤
XlinkController.resolveManifestOnLoad -> onXlinkAllElementsLoaded -> Events.INTERNAL_MANIFEST_LOADED
Events.INTERNAL_MANIFEST_LOADED -> StreamController.onManifestUpdated -> TimeSyncController.initialize -> TimeSyncController.attemptSync -> Events.TIME_SYNCHRONIZATION_COMPLETED
StreamController.onTimeSyncCompleted -> composeStreams -> switchStream -> openMediaSource or preloadStream -> onMediaSourceOpen -> Events.SOURCE_INITIALIZED -> activateStream
StreamController.activateStream -> Stream.activate -> initializeMedia -> checkIfInitializationCompleted -> Events.STREAM_INITIALIZED
Events.STREAM_INITIALIZED -> ScheduleController.onStreamInitialized -> start -> startScheduleTimer(0) -> schedule -> getNextFragment -> switchInitData
SourceBufferSink.updateEndHandler -> afterSuccess -> BufferController.onAppended -> ScheduleController.onBytesAppended -> startScheduleTimer

# 更新 mpd
StreamController.activateStream -> PlaybackController.play -> onPlaybackStart -> Events.PLAYBACK_STARTED ->
ManifestUpdater.onPlaybackStarted -> startManifestRefreshTimer -> onRefreshTimer -> refreshManifest -> 
XlinkController.resolveManifestOnLoad -> onXlinkAllElementsLoaded -> ManifestUpdater.onManifestLoaded -> update -> startManifestRefreshTimer

# mpd 更新后操作
请求 mpd -> PlaybackController.onDataUpdateCompleted -> updateCurrentTime -> getActualPresentationTime
currentTime = video.currentTime
actualTime = PlaybackController.getActualPresentationTime
这里会对比当前播放器的 currentTime 与实际应该播放的时间，如果不一致就会执行 PlaybackController.seek(actualTime)

PlaybackController.getActualPresentationTime -> DashMetrics.getCurrentDVRInfo -> list\[list.length - 1\]

DVRInfo 的添加过程：
1. video(progress event) -> PlaybackController.onPlaybackProgress
2. OnFragmentLoadingCompleted -> SourceBufferSink.updateEndHandler
3. video(timeupdate event) -> PlaybackController.onPlaybackTimeUpdated
-> BufferController.onPlaybackProgression -> BufferController(Events.BUFFER_LEVEL_UPDATED) ->
StreamProcessor.onBufferLevelUpdated -> addDVRMetric(TimelineConverter.calcSegmentAvailabilityRange) -> DashMetrics.addDVRInfo ->
MetricsModel.addDVRInfo

# 延迟计算
PlaybackController.computeLiveDelay

# 名次解释
streamInfo is a alias for Period
media info is a alias for AdaptationSet

DVRWindow(TimelineConverter.calcSegmentAvailabilityRange): = segmentAvailabilityRange, 计算的 segment 可用范围, 主要是根据 Period.start, Period.duration, mpd.timeShiftBufferDepth, now(要计算与服务器的偏移), Representation.availabilityTimeOffset 等计算
liveEdge: = representationInfo.DVRWindow.end, 直播边缘
DVRWindowSize: = timeShiftBufferDepth
liveDelay(PlaybackController.computeLiveDelay): lowLatencyEnabled = 0 > setting > fragmentDuration * liveDelayFragmentCount > uriParameters.r > suggestedPresentationDelay > fragmentDuration * 4 > minBufferTime * 4,
if set timeShiftBufferDepth, targetDelayCapping = Math.max(dvrWindowSize - END_OF_PLAYLIST_PADDING, dvrWindowSize / 2), then Math.min(delay, targetDelayCapping)
