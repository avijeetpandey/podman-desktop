<script lang="ts">
import { faArrowUp, faLayerGroup, faPlay, faTrash, faEdit } from '@fortawesome/free-solid-svg-icons';
import type { ImageInfoUI } from './ImageInfoUI';
import { router } from 'tinro';
import ListItemButtonIcon from '../ui/ListItemButtonIcon.svelte';
import { runImageInfo } from '../../stores/run-image-store';
import type { Menu } from '../../../../main/src/plugin/menu-registry';
import ContributionActions from '/@/lib/actions/ContributionActions.svelte';
import { ImageUtils } from './image-utils';
import { createEventDispatcher, onDestroy, onMount } from 'svelte';
import { MenuContext } from '../../../../main/src/plugin/menu-registry';
import ActionsWrapper from './ActionsMenu.svelte';
import type { Unsubscriber } from 'svelte/motion';
import { ContextUI } from '../context/context';
import { context } from '/@/stores/context';

export let onPushImage: (imageInfo: ImageInfoUI) => void;
export let onRenameImage: (imageInfo: ImageInfoUI) => void;
export let image: ImageInfoUI;
export let dropdownMenu = false;
export let detailed = false;
export let groupContributions = false;

let isAuthenticatedForThisImage = false;
const imageUtils = new ImageUtils();

let contributions: Menu[] = [];
let globalContext: ContextUI;
let contextsUnsubscribe: Unsubscriber;
let groupingContributions = false;

const dispatch = createEventDispatcher<{ update: ImageInfoUI }>();

onMount(async () => {
  contributions = await window.getContributedMenus(MenuContext.DASHBOARD_IMAGE);
  groupingContributions = groupContributions && !dropdownMenu && contributions.length > 1;
  contextsUnsubscribe = context.subscribe(value => {
    // Copy context, do not use reference
    globalContext = new ContextUI();
    const allValues = value.collectAllValues();
    for (const k in allValues) {
      globalContext.setValue(k, allValues[k]);
    }

    const labels = image.labels ? Object.keys(image.labels) : [];
    globalContext.setValue('imageLabelKeys', labels);
  });
});

onDestroy(() => {
  // unsubscribe from the store
  if (contextsUnsubscribe) {
    contextsUnsubscribe();
  }
});

async function runImage(imageInfo: ImageInfoUI) {
  runImageInfo.set(imageInfo);
  router.goto('/images/run/basic');
}

$: window.hasAuthconfigForImage(image.name).then(result => (isAuthenticatedForThisImage = result));

async function deleteImage(): Promise<void> {
  image.status = 'DELETING';
  dispatch('update', image);

  try {
    await imageUtils.deleteImage(image);
  } catch (error) {
    onError(`Error while deleting image: ${String(error)}`);
  }
}

async function renameImage(imageInfo: ImageInfoUI): Promise<void> {
  onRenameImage(imageInfo);
}

async function pushImage(imageInfo: ImageInfoUI): Promise<void> {
  onPushImage(imageInfo);
}

async function showLayersImage(): Promise<void> {
  router.goto(`/images/${image.id}/${image.engineId}/${image.base64RepoTag}/history`);
}

function onError(error: string): void {
  window.showMessageBox({
    title: 'Something went wrong.',
    message: error,
    type: 'error',
  });
}
</script>

<ListItemButtonIcon title="Run Image" onClick="{() => runImage(image)}" detailed="{detailed}" icon="{faPlay}" />

<ListItemButtonIcon
  title="Delete Image"
  confirm="{true}"
  onClick="{() => deleteImage()}"
  detailed="{detailed}"
  icon="{faTrash}"
  enabled="{image.status === 'UNUSED'}" />

<!-- If dropdownMenu is true, use it, otherwise just show the regular buttons -->
<ActionsWrapper
  dropdownMenu="{dropdownMenu}"
  onBeforeToggle="{() => {
    globalContext?.setValue('selectedImageId', image.id);
  }}">
  {#if isAuthenticatedForThisImage}
    <ListItemButtonIcon
      title="Push Image"
      onClick="{() => pushImage(image)}"
      menu="{dropdownMenu}"
      detailed="{detailed}"
      icon="{faArrowUp}" />
  {/if}

  <ListItemButtonIcon
    title="Edit Image"
    onClick="{() => renameImage(image)}"
    menu="{dropdownMenu}"
    detailed="{detailed}"
    icon="{faEdit}" />

  {#if !detailed}
    <ListItemButtonIcon
      title="Show History"
      onClick="{() => showLayersImage()}"
      menu="{dropdownMenu}"
      detailed="{detailed}"
      icon="{faLayerGroup}" />
  {/if}

  <ActionsWrapper dropdownMenu="{groupingContributions}" dropdownMenuAsMenuActionItem="{groupingContributions}">
    <ContributionActions
      args="{[image]}"
      dropdownMenu="{groupingContributions ? true : dropdownMenu}"
      contributions="{contributions}"
      contextPrefix="imageItem"
      detailed="{detailed}"
      onError="{onError}"
      contextUI="{globalContext}" />
  </ActionsWrapper>
</ActionsWrapper>
