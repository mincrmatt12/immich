<script lang="ts">
  import { focusTrap } from '$lib/actions/focus-trap';
  import type { Action, OnAction } from '$lib/components/asset-viewer/actions/action';
  import MotionPhotoAction from '$lib/components/asset-viewer/actions/motion-photo-action.svelte';
  import NextAssetAction from '$lib/components/asset-viewer/actions/next-asset-action.svelte';
  import PreviousAssetAction from '$lib/components/asset-viewer/actions/previous-asset-action.svelte';
  import Icon from '$lib/components/elements/icon.svelte';
  import { AssetAction, ProjectionType } from '$lib/constants';
  import { updateNumberOfComments } from '$lib/stores/activity.store';
  import { assetViewingStore } from '$lib/stores/asset-viewing.store';
  import type { AssetStore } from '$lib/stores/assets.store';
  import { isShowDetail } from '$lib/stores/preferences.store';
  import { SlideshowNavigation, SlideshowState, slideshowStore } from '$lib/stores/slideshow.store';
  import { user } from '$lib/stores/user.store';
  import { websocketEvents } from '$lib/stores/websocket';
  import { getAssetJobMessage, getSharedLink, handlePromiseError, isSharedLink } from '$lib/utils';
  import { handleError } from '$lib/utils/handle-error';
  import { navigate } from '$lib/utils/navigation';
  import { SlideshowHistory } from '$lib/utils/slideshow-history';
  import {
    AssetJobName,
    AssetTypeEnum,
    ReactionType,
    createActivity,
    deleteActivity,
    getActivities,
    getActivityStatistics,
    getAllAlbums,
    runAssetJobs,
    type ActivityResponseDto,
    type AlbumResponseDto,
    type AssetResponseDto,
  } from '@immich/sdk';
  import { mdiImageBrokenVariant } from '@mdi/js';
  import { createEventDispatcher, onDestroy, onMount } from 'svelte';
  import { t } from 'svelte-i18n';
  import { fly } from 'svelte/transition';
  import Thumbnail from '../assets/thumbnail/thumbnail.svelte';
  import { NotificationType, notificationController } from '../shared-components/notification/notification';
  import ActivityStatus from './activity-status.svelte';
  import ActivityViewer from './activity-viewer.svelte';
  import AssetViewerNavBar from './asset-viewer-nav-bar.svelte';
  import DetailPanel from './detail-panel.svelte';
  import PanoramaViewer from './panorama-viewer.svelte';
  import PhotoViewer from './photo-viewer.svelte';
  import SlideshowBar from './slideshow-bar.svelte';
  import VideoViewer from './video-wrapper-viewer.svelte';

  export let assetStore: AssetStore | null = null;
  export let asset: AssetResponseDto;
  export let preloadAssets: AssetResponseDto[] = [];
  export let showNavigation = true;
  export let withStacked = false;
  export let isShared = false;
  export let album: AlbumResponseDto | null = null;
  export let onAction: OnAction | undefined = undefined;

  let reactions: ActivityResponseDto[] = [];

  const { setAsset } = assetViewingStore;
  const {
    restartProgress: restartSlideshowProgress,
    stopProgress: stopSlideshowProgress,
    slideshowNavigation,
    slideshowState,
  } = slideshowStore;

  const dispatch = createEventDispatcher<{
    close: void;
    next: void;
    previous: void;
  }>();

  let appearsInAlbums: AlbumResponseDto[] = [];
  let stackedAssets: AssetResponseDto[] = [];
  let shouldPlayMotionPhoto = false;
  let sharedLink = getSharedLink();
  let enableDetailPanel = asset.hasMetadata;
  let slideshowStateUnsubscribe: () => void;
  let shuffleSlideshowUnsubscribe: () => void;
  let previewStackedAsset: AssetResponseDto | undefined;
  let isShowActivity = false;
  let isLiked: ActivityResponseDto | null = null;
  let numberOfComments: number;
  let fullscreenElement: Element;
  let unsubscribe: () => void;
  let zoomToggle = () => void 0;
  let copyImage: () => Promise<void>;

  $: isFullScreen = fullscreenElement !== null;

  $: {
    if (asset.stackCount && asset.stack) {
      stackedAssets = asset.stack;
      stackedAssets = [...stackedAssets, asset].sort(
        (a, b) => new Date(b.fileCreatedAt).getTime() - new Date(a.fileCreatedAt).getTime(),
      );

      // if its a stack, add the next stack image in addition to the next asset
      if (asset.stackCount > 1) {
        preloadAssets.push(stackedAssets[1]);
      }
    }

    if (!stackedAssets.map((a) => a.id).includes(asset.id)) {
      stackedAssets = [];
    }
  }

  $: {
    if (album && !album.isActivityEnabled && numberOfComments === 0) {
      isShowActivity = false;
    }
  }

  const handleAddComment = () => {
    numberOfComments++;
    updateNumberOfComments(1);
  };

  const handleRemoveComment = () => {
    numberOfComments--;
    updateNumberOfComments(-1);
  };

  const handleFavorite = async () => {
    if (album && album.isActivityEnabled) {
      try {
        if (isLiked) {
          const activityId = isLiked.id;
          await deleteActivity({ id: activityId });
          reactions = reactions.filter((reaction) => reaction.id !== activityId);
          isLiked = null;
        } else {
          const data = await createActivity({
            activityCreateDto: { albumId: album.id, assetId: asset.id, type: ReactionType.Like },
          });

          isLiked = data;
          reactions = [...reactions, isLiked];
        }
      } catch (error) {
        handleError(error, $t('errors.unable_to_change_favorite'));
      }
    }
  };

  const getFavorite = async () => {
    if (album && $user) {
      try {
        const data = await getActivities({
          userId: $user.id,
          assetId: asset.id,
          albumId: album.id,
          $type: ReactionType.Like,
        });
        isLiked = data.length > 0 ? data[0] : null;
      } catch (error) {
        handleError(error, $t('errors.unable_to_load_liked_status'));
      }
    }
  };

  const getNumberOfComments = async () => {
    if (album) {
      try {
        const { comments } = await getActivityStatistics({ assetId: asset.id, albumId: album.id });
        numberOfComments = comments;
      } catch (error) {
        handleError(error, $t('errors.unable_to_get_comments_number'));
      }
    }
  };

  $: {
    if (isShared && asset.id) {
      handlePromiseError(getFavorite());
      handlePromiseError(getNumberOfComments());
    }
  }

  onMount(async () => {
    unsubscribe = websocketEvents.on('on_upload_success', (assetUpdate) => {
      if (assetUpdate.id === asset.id) {
        asset = assetUpdate;
      }
    });
    await navigate({ targetRoute: 'current', assetId: asset.id });
    slideshowStateUnsubscribe = slideshowState.subscribe((value) => {
      if (value === SlideshowState.PlaySlideshow) {
        slideshowHistory.reset();
        slideshowHistory.queue(asset);
        handlePromiseError(handlePlaySlideshow());
      } else if (value === SlideshowState.StopSlideshow) {
        handlePromiseError(handleStopSlideshow());
      }
    });

    shuffleSlideshowUnsubscribe = slideshowNavigation.subscribe((value) => {
      if (value === SlideshowNavigation.Shuffle) {
        slideshowHistory.reset();
        slideshowHistory.queue(asset);
      }
    });

    if (!sharedLink) {
      await handleGetAllAlbums();
    }

    if (asset.stackCount && asset.stack) {
      stackedAssets = asset.stack;
      stackedAssets = [...stackedAssets, asset].sort(
        (a, b) => new Date(a.fileCreatedAt).getTime() - new Date(b.fileCreatedAt).getTime(),
      );
    } else {
      stackedAssets = [];
    }
  });

  onDestroy(() => {
    if (slideshowStateUnsubscribe) {
      slideshowStateUnsubscribe();
    }

    if (shuffleSlideshowUnsubscribe) {
      shuffleSlideshowUnsubscribe();
    }
    unsubscribe?.();
  });

  $: {
    if (asset.id && !sharedLink) {
      handlePromiseError(handleGetAllAlbums());
    }
  }

  const handleGetAllAlbums = async () => {
    if (isSharedLink()) {
      return;
    }

    try {
      appearsInAlbums = await getAllAlbums({ assetId: asset.id });
    } catch (error) {
      console.error('Error getting album that asset belong to', error);
    }
  };

  const handleOpenActivity = () => {
    if ($isShowDetail) {
      $isShowDetail = false;
    }
    isShowActivity = !isShowActivity;
  };

  const toggleDetailPanel = () => {
    isShowActivity = false;
    $isShowDetail = !$isShowDetail;
  };

  const closeViewer = async () => {
    dispatch('close');
    await navigate({ targetRoute: 'current', assetId: null });
  };

  const navigateAssetRandom = async () => {
    if (!assetStore) {
      return;
    }

    const asset = await assetStore.getRandomAsset();
    if (!asset) {
      return;
    }

    slideshowHistory.queue(asset);

    setAsset(asset);
    $restartSlideshowProgress = true;
  };

  const navigateAsset = async (order?: 'previous' | 'next', e?: Event) => {
    if (!order) {
      if ($slideshowState === SlideshowState.PlaySlideshow) {
        order = $slideshowNavigation === SlideshowNavigation.AscendingOrder ? 'previous' : 'next';
      } else {
        return;
      }
    }

    if ($slideshowState === SlideshowState.PlaySlideshow && $slideshowNavigation === SlideshowNavigation.Shuffle) {
      return (order === 'previous' ? slideshowHistory.previous() : slideshowHistory.next()) || navigateAssetRandom();
    }

    if ($slideshowState === SlideshowState.PlaySlideshow && assetStore) {
      const hasNext =
        order === 'previous' ? await assetStore.getPreviousAsset(asset) : await assetStore.getNextAsset(asset);
      if (hasNext) {
        $restartSlideshowProgress = true;
      } else {
        await handleStopSlideshow();
      }
    }

    e?.stopPropagation();
    dispatch(order);
  };

  const handleRunJob = async (name: AssetJobName) => {
    try {
      await runAssetJobs({ assetJobsDto: { assetIds: [asset.id], name } });
      notificationController.show({ type: NotificationType.Info, message: $getAssetJobMessage(name) });
    } catch (error) {
      handleError(error, $t('errors.unable_to_submit_job'));
    }
  };

  /**
   * Slide show mode
   */

  let assetViewerHtmlElement: HTMLElement;

  const slideshowHistory = new SlideshowHistory((asset) => {
    setAsset(asset);
    $restartSlideshowProgress = true;
  });

  const handleVideoStarted = () => {
    if ($slideshowState === SlideshowState.PlaySlideshow) {
      $stopSlideshowProgress = true;
    }
  };

  const handlePlaySlideshow = async () => {
    try {
      await assetViewerHtmlElement.requestFullscreen();
    } catch (error) {
      handleError(error, $t('errors.unable_to_enter_fullscreen'));
      $slideshowState = SlideshowState.StopSlideshow;
    }
  };

  const handleStopSlideshow = async () => {
    try {
      if (document.fullscreenElement) {
        document.body.style.cursor = '';
        await document.exitFullscreen();
      }
    } catch (error) {
      handleError(error, $t('errors.unable_to_exit_fullscreen'));
    } finally {
      $stopSlideshowProgress = true;
      $slideshowState = SlideshowState.None;
    }
  };

  const handleStackedAssetMouseEvent = (e: CustomEvent<{ isMouseOver: boolean }>, asset: AssetResponseDto) => {
    const { isMouseOver } = e.detail;

    previewStackedAsset = isMouseOver ? asset : undefined;
  };

  const handleAction = async (action: Action) => {
    switch (action.type) {
      case AssetAction.ADD_TO_ALBUM: {
        await handleGetAllAlbums();
        break;
      }
      case AssetAction.UNSTACK: {
        await closeViewer();
      }
    }

    onAction?.(action);
  };
</script>

<svelte:document bind:fullscreenElement />

<section
  id="immich-asset-viewer"
  class="fixed left-0 top-0 z-[1001] grid h-screen w-screen grid-cols-4 grid-rows-[64px_1fr] overflow-hidden bg-black"
  use:focusTrap
>
  <!-- Top navigation bar -->
  {#if $slideshowState === SlideshowState.None}
    <div class="z-[1002] col-span-4 col-start-1 row-span-1 row-start-1 transition-transform">
      <AssetViewerNavBar
        {asset}
        {album}
        {stackedAssets}
        showDetailButton={enableDetailPanel}
        showSlideshow={!!assetStore}
        hasStackChildren={stackedAssets.length > 0}
        onZoomImage={zoomToggle}
        onCopyImage={copyImage}
        onAction={handleAction}
        onRunJob={handleRunJob}
        onPlaySlideshow={() => ($slideshowState = SlideshowState.PlaySlideshow)}
        onShowDetail={toggleDetailPanel}
        onClose={closeViewer}
      >
        <MotionPhotoAction
          slot="motion-photo"
          isPlaying={shouldPlayMotionPhoto}
          onClick={(shouldPlay) => (shouldPlayMotionPhoto = shouldPlay)}
        />
      </AssetViewerNavBar>
    </div>
  {/if}

  {#if $slideshowState === SlideshowState.None && showNavigation}
    <div class="z-[1001] my-auto column-span-1 col-start-1 row-span-full row-start-1 justify-self-start">
      <PreviousAssetAction onPreviousAsset={() => navigateAsset('previous')} />
    </div>
  {/if}

  <!-- Asset Viewer -->
  <div class="z-[1000] relative col-start-1 col-span-4 row-start-1 row-span-full" bind:this={assetViewerHtmlElement}>
    {#if $slideshowState != SlideshowState.None}
      <div class="z-[1000] absolute w-full flex">
        <SlideshowBar
          {isFullScreen}
          onSetToFullScreen={() => assetViewerHtmlElement.requestFullscreen()}
          onPrevious={() => navigateAsset('previous')}
          onNext={() => navigateAsset('next')}
          onClose={() => ($slideshowState = SlideshowState.StopSlideshow)}
        />
      </div>
    {/if}

    {#if previewStackedAsset}
      {#key previewStackedAsset.id}
        {#if previewStackedAsset.type === AssetTypeEnum.Image}
          <PhotoViewer
            bind:zoomToggle
            bind:copyImage
            asset={previewStackedAsset}
            {preloadAssets}
            on:close={closeViewer}
            haveFadeTransition={false}
            {sharedLink}
          />
        {:else}
          <VideoViewer
            assetId={previewStackedAsset.id}
            checksum={previewStackedAsset.checksum}
            projectionType={previewStackedAsset.exifInfo?.projectionType}
            loopVideo={true}
            on:close={closeViewer}
            on:onVideoEnded={() => navigateAsset()}
            on:onVideoStarted={handleVideoStarted}
          />
        {/if}
      {/key}
    {:else}
      {#key asset.id}
        {#if !asset.resized}
          <div class="flex h-full w-full justify-center">
            <div
              class="px-auto flex aspect-square h-full items-center justify-center bg-gray-100 dark:bg-immich-dark-gray"
            >
              <Icon path={mdiImageBrokenVariant} size="25%" />
            </div>
          </div>
        {:else if asset.type === AssetTypeEnum.Image}
          {#if shouldPlayMotionPhoto && asset.livePhotoVideoId}
            <VideoViewer
              assetId={asset.livePhotoVideoId}
              checksum={asset.checksum}
              projectionType={asset.exifInfo?.projectionType}
              loopVideo={$slideshowState !== SlideshowState.PlaySlideshow}
              on:close={closeViewer}
              on:onVideoEnded={() => (shouldPlayMotionPhoto = false)}
            />
          {:else if asset.exifInfo?.projectionType === ProjectionType.EQUIRECTANGULAR || (asset.originalPath && asset.originalPath
                .toLowerCase()
                .endsWith('.insp'))}
            <PanoramaViewer {asset} />
          {:else}
            <PhotoViewer bind:zoomToggle bind:copyImage {asset} {preloadAssets} on:close={closeViewer} {sharedLink} />
          {/if}
        {:else}
          <VideoViewer
            assetId={asset.id}
            checksum={asset.checksum}
            projectionType={asset.exifInfo?.projectionType}
            loopVideo={$slideshowState !== SlideshowState.PlaySlideshow}
            on:close={closeViewer}
            on:onVideoEnded={() => navigateAsset()}
            on:onVideoStarted={handleVideoStarted}
          />
        {/if}
        {#if $slideshowState === SlideshowState.None && isShared && ((album && album.isActivityEnabled) || numberOfComments > 0)}
          <div class="z-[9999] absolute bottom-0 right-0 mb-20 mr-8">
            <ActivityStatus
              disabled={!album?.isActivityEnabled}
              {isLiked}
              {numberOfComments}
              on:favorite={handleFavorite}
              on:openActivityTab={handleOpenActivity}
            />
          </div>
        {/if}
      {/key}
    {/if}
  </div>

  {#if $slideshowState === SlideshowState.None && showNavigation}
    <div class="z-[1001] my-auto col-span-1 col-start-4 row-span-full row-start-1 justify-self-end">
      <NextAssetAction onNextAsset={() => navigateAsset('next')} />
    </div>
  {/if}

  {#if enableDetailPanel && $slideshowState === SlideshowState.None && $isShowDetail}
    <div
      transition:fly={{ duration: 150 }}
      id="detail-panel"
      class="z-[1002] row-start-1 row-span-4 w-[360px] overflow-y-auto bg-immich-bg transition-all dark:border-l dark:border-l-immich-dark-gray dark:bg-immich-dark-bg"
      translate="yes"
    >
      <DetailPanel {asset} currentAlbum={album} albums={appearsInAlbums} on:close={() => ($isShowDetail = false)} />
    </div>
  {/if}

  {#if stackedAssets.length > 0 && withStacked}
    <div
      id="stack-slideshow"
      class="z-[1002] flex place-item-center place-content-center absolute bottom-0 w-full col-span-4 col-start-1 overflow-x-auto horizontal-scrollbar"
    >
      <div class="relative w-full whitespace-nowrap transition-all">
        {#each stackedAssets as stackedAsset, index (stackedAsset.id)}
          <div
            class="{stackedAsset.id == asset.id
              ? '-translate-y-[1px]'
              : '-translate-y-0'} inline-block px-1 transition-transform"
          >
            <Thumbnail
              class="{stackedAsset.id == asset.id
                ? 'bg-transparent border-2 border-white'
                : 'bg-gray-700/40'} inline-block hover:bg-transparent"
              asset={stackedAsset}
              onClick={(stackedAsset, event) => {
                event.preventDefault();
                asset = stackedAsset;
                preloadAssets = index + 1 >= stackedAssets.length ? [] : [stackedAssets[index + 1]];
              }}
              on:mouse-event={(e) => handleStackedAssetMouseEvent(e, stackedAsset)}
              readonly
              thumbnailSize={stackedAsset.id == asset.id ? 65 : 60}
              showStackedIcon={false}
            />

            {#if stackedAsset.id == asset.id}
              <div class="w-full flex place-items-center place-content-center">
                <div class="w-2 h-2 bg-white rounded-full flex mt-[2px]" />
              </div>
            {/if}
          </div>
        {/each}
      </div>
    </div>
  {/if}

  {#if isShared && album && isShowActivity && $user}
    <div
      transition:fly={{ duration: 150 }}
      id="activity-panel"
      class="z-[1002] row-start-1 row-span-5 w-[360px] md:w-[460px] overflow-y-auto bg-immich-bg transition-all dark:border-l dark:border-l-immich-dark-gray dark:bg-immich-dark-bg"
      translate="yes"
    >
      <ActivityViewer
        user={$user}
        disabled={!album.isActivityEnabled}
        assetType={asset.type}
        albumOwnerId={album.ownerId}
        albumId={album.id}
        assetId={asset.id}
        {isLiked}
        bind:reactions
        on:addComment={handleAddComment}
        on:deleteComment={handleRemoveComment}
        on:deleteLike={() => (isLiked = null)}
        on:close={() => (isShowActivity = false)}
      />
    </div>
  {/if}
</section>

<style>
  #immich-asset-viewer {
    contain: layout;
  }

  .horizontal-scrollbar::-webkit-scrollbar {
    width: 8px;
    height: 10px;
  }

  /* Track */
  .horizontal-scrollbar::-webkit-scrollbar-track {
    background: #000000;
    border-radius: 16px;
  }

  /* Handle */
  .horizontal-scrollbar::-webkit-scrollbar-thumb {
    background: rgba(159, 159, 159, 0.408);
    border-radius: 16px;
  }

  /* Handle on hover */
  .horizontal-scrollbar::-webkit-scrollbar-thumb:hover {
    background: #adcbfa;
    border-radius: 16px;
  }
</style>
