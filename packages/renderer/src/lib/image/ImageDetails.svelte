<script lang="ts">
import type { ImageInfoUI } from './ImageInfoUI';
import Route from '../../Route.svelte';
import { onDestroy, onMount } from 'svelte';
import { imagesInfos } from '../../stores/images';
import StatusIcon from '../images/StatusIcon.svelte';
import ImageActions from './ImageActions.svelte';
import { ImageUtils } from './image-utils';
import ImageDetailsInspect from './ImageDetailsInspect.svelte';
import ImageDetailsHistory from './ImageDetailsHistory.svelte';
import ImageDetailsSummary from './ImageDetailsSummary.svelte';
import PushImageModal from './PushImageModal.svelte';
import RenameImageModal from './RenameImageModal.svelte';
import DetailsPage from '../ui/DetailsPage.svelte';
import Tab from '../ui/Tab.svelte';
import { containersInfos } from '/@/stores/containers';
import ImageDetailsCheck from './ImageDetailsCheck.svelte';
import { imageCheckerProviders } from '/@/stores/image-checker-providers';
import type { Unsubscriber } from 'svelte/motion';
import type { ImageInfo } from '@podman-desktop/api';
import { viewsContributions } from '/@/stores/views';
import type { ViewInfoUI } from '../../../../main/src/plugin/api/view-info';
import {
  IMAGE_DETAILS_VIEW_BADGES,
  IMAGE_DETAILS_VIEW_ICONS,
  IMAGE_VIEW_BADGES,
  IMAGE_VIEW_ICONS,
} from '../view/views';
import { context } from '/@/stores/context';
import type { ContextUI } from '../context/context';
import Badge from '../ui/Badge.svelte';

export let imageID: string;
export let engineId: string;
export let base64RepoTag: string;

let globalContext: ContextUI;
let viewContributions: ViewInfoUI[] = [];
let allImages: ImageInfo[];

const imageUtils = new ImageUtils();

let pushImageModal = false;
function handlePushImageModal() {
  pushImageModal = true;
}

let renameImageModal = false;
function handleRenameImageModal() {
  renameImageModal = true;
}

function closeModals() {
  pushImageModal = false;
  renameImageModal = false;
}

let imageInfo: ImageInfo | undefined;
let image: ImageInfoUI;
let detailsPage: DetailsPage;

let showCheckTab: boolean = false;
let providersUnsubscribe: Unsubscriber;
let viewsUnsubscribe: Unsubscriber;
let contextsUnsubscribe: Unsubscriber;

function updateImage() {
  if (!allImages) {
    return;
  }
  imageInfo = allImages.find(c => c.Id === imageID && c.engineId === engineId);
  let tempImage;
  if (imageInfo) {
    tempImage = imageUtils.getImageInfoUI(imageInfo, base64RepoTag, $containersInfos, globalContext, viewContributions);
  }
  if (tempImage) {
    image = tempImage;
  } else {
    // the image has been deleted
    detailsPage.close();
  }
}

onMount(() => {
  providersUnsubscribe = imageCheckerProviders.subscribe(providers => {
    showCheckTab = providers.length > 0;
  });

  viewsUnsubscribe = viewsContributions.subscribe(value => {
    viewContributions =
      value.filter(
        view =>
          view.viewId === IMAGE_DETAILS_VIEW_ICONS ||
          view.viewId === IMAGE_VIEW_ICONS ||
          view.viewId === IMAGE_VIEW_BADGES ||
          view.viewId === IMAGE_DETAILS_VIEW_BADGES,
      ) || [];
    updateImage();
  });

  contextsUnsubscribe = context.subscribe(value => {
    globalContext = value;
    updateImage();
  });

  // loading image info
  return imagesInfos.subscribe(images => {
    allImages = images;
    updateImage();
  });
});

onDestroy(() => {
  // unsubscribe from the store
  providersUnsubscribe?.();
  viewsUnsubscribe?.();
  contextsUnsubscribe?.();
});
</script>

{#if image}
  <DetailsPage title="{image.name}" titleDetail="{image.shortId}" subtitle="{image.tag}" bind:this="{detailsPage}">
    <StatusIcon slot="icon" icon="{image.icon}" size="{24}" status="{image.status}" />
    <svelte:fragment slot="subtitle">
      {#if image.badges.length}
        <div class="flex flex-row">
          {#each image.badges as badge}
            <Badge color="{badge.color}" label="{badge.label}" />
          {/each}
        </div>
      {/if}
    </svelte:fragment>
    <ImageActions
      slot="actions"
      image="{image}"
      onPushImage="{handlePushImageModal}"
      onRenameImage="{handleRenameImageModal}"
      detailed="{true}"
      dropdownMenu="{false}"
      groupContributions="{true}"
      on:update="{() => (image = image)}" />
    <svelte:fragment slot="tabs">
      <Tab title="Summary" url="summary" />
      <Tab title="History" url="history" />
      <Tab title="Inspect" url="inspect" />
      {#if showCheckTab}
        <Tab title="Check" url="check" />
      {/if}
    </svelte:fragment>
    <svelte:fragment slot="content">
      <Route path="/summary" breadcrumb="Summary" navigationHint="tab">
        <ImageDetailsSummary image="{image}" />
      </Route>
      <Route path="/history" breadcrumb="History" navigationHint="tab">
        <ImageDetailsHistory image="{image}" />
      </Route>
      <Route path="/inspect" breadcrumb="Inspect" navigationHint="tab">
        <ImageDetailsInspect image="{image}" />
      </Route>
      <Route path="/check" breadcrumb="Check" navigationHint="tab">
        <ImageDetailsCheck imageInfo="{imageInfo}" />
      </Route>
    </svelte:fragment>
  </DetailsPage>
{/if}

{#if pushImageModal}
  <PushImageModal
    imageInfoToPush="{image}"
    closeCallback="{() => {
      closeModals();
    }}" />
{/if}
{#if renameImageModal}
  <RenameImageModal
    imageInfoToRename="{image}"
    detailed="{true}"
    closeCallback="{() => {
      closeModals();
    }}" />
{/if}
