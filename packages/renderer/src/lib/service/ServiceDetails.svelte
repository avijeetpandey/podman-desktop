<script lang="ts">
import Route from '../../Route.svelte';
import { onMount } from 'svelte';
import StatusIcon from '../images/StatusIcon.svelte';
import DetailsPage from '../ui/DetailsPage.svelte';
import Tab from '../ui/Tab.svelte';
import StateChange from '../ui/StateChange.svelte';
import type { V1Service } from '@kubernetes/client-node';
import { stringify } from 'yaml';
import MonacoEditor from '../editor/MonacoEditor.svelte';
import type { ServiceUI } from './ServiceUI';
import { ServiceUtils } from './service-utils';
import { services } from '/@/stores/services';
import ServiceActions from './ServiceActions.svelte';
import ServiceDetailsSummary from './ServiceDetailsSummary.svelte';
import ServiceIcon from '../images/ServiceIcon.svelte';

export let name: string;
export let namespace: string;

let service: ServiceUI;
let detailsPage: DetailsPage;
let kubeService: V1Service | undefined;
let kubeError: string;

onMount(() => {
  const serviceUtils = new ServiceUtils();
  // loading service info
  return services.subscribe(services => {
    const matchingService = services.find(srv => srv.metadata?.name === name && srv.metadata?.namespace === namespace);
    if (matchingService) {
      try {
        service = serviceUtils.getServiceUI(matchingService);
        loadDetails();
      } catch (err) {
        console.error(err);
      }
    } else if (detailsPage) {
      // the service has been deleted
      detailsPage.close();
    }
  });
});

async function loadDetails() {
  const getKubeService = await window.kubernetesReadNamespacedService(service.name, namespace);
  if (getKubeService) {
    kubeService = getKubeService;
  } else {
    kubeError = `Unable to retrieve Kubernetes details for ${service.name}`;
  }
}
</script>

{#if service}
  <DetailsPage title="{service.name}" subtitle="{service.namespace}" bind:this="{detailsPage}">
    <StatusIcon slot="icon" icon="{ServiceIcon}" size="{24}" status="{service.status}" />
    <svelte:fragment slot="actions">
      <ServiceActions service="{service}" detailed="{true}" on:update="{() => (service = service)}" />
    </svelte:fragment>
    <div slot="detail" class="flex py-2 w-full justify-end text-sm text-gray-700">
      <StateChange state="{service.status}" />
    </div>
    <svelte:fragment slot="tabs">
      <Tab title="Summary" url="summary" />
      <Tab title="Inspect" url="inspect" />
      <Tab title="Kube" url="kube" />
    </svelte:fragment>
    <svelte:fragment slot="content">
      <Route path="/summary" breadcrumb="Summary" navigationHint="tab">
        <ServiceDetailsSummary serviceUI="{service}" service="{kubeService}" kubeError="{kubeError}" />
      </Route>
      <Route path="/inspect" breadcrumb="Inspect" navigationHint="tab">
        <MonacoEditor content="{JSON.stringify(kubeService, undefined, 2)}" language="json" />
      </Route>
      <Route path="/kube" breadcrumb="Kube" navigationHint="tab">
        <MonacoEditor content="{stringify(kubeService)}" language="yaml" />
      </Route>
    </svelte:fragment>
  </DetailsPage>
{/if}
