<script lang="ts">
  import { Handle, Position } from '@xyflow/svelte';
  
  export let id: string;
  export let data: any;
  export let type: string;
  export let selected: boolean = false;
  export let dragging: boolean = false;
  export let draggable: boolean = true;
  export let selectable: boolean = true;
  export let deletable: boolean = true;
  export let zIndex: number = 0;

  // svelteflow props
  export let isConnectable: boolean = true;
  export let positionAbsoluteX: number = 0;
  export let positionAbsoluteY: number = 0;
  export let sourcePosition: any = undefined;
  export let targetPosition: any = undefined;
  export let width: number | undefined = undefined;
  export let height: number | undefined = undefined;
  export let dragHandle: string | undefined = undefined;
  export let parentId: string | undefined = undefined;
  export let parentNode: string | undefined = undefined;
  export let xPos: number = 0;
  export let yPos: number = 0;
  export let connectable: boolean = true;
</script>

<div class="leaf-node-content" class:selected>
  <!-- handle for metadata output  -->
  <Handle 
    type="target" 
    position={Position.Right} 
    id={`${id}-handle`} 
    style="position: absolute; right: -4px; top: 50%; transform: translateY(-50%); width: 8px; height: 8px; background: #888; border: 2px solid white; border-radius: 50%;"
  />

  <div class="leaf-content">
    <div class="leaf-label">{data?.label || 'Metadata Field'}</div>
    {#if data?.type}
      <div class="leaf-type">{data.type}</div>
    {/if}
    {#if data?.path}
      <div class="leaf-path">{data.path}</div>
    {/if}
  </div>
</div>

<style>
  .leaf-node-content {
    border: 2px solid #888;
    border-radius: 6px;
    background: #f8f8f8;
    padding: 8px 12px;
    min-width: 120px;
    position: relative;
    transition: all 0.2s;
  }
  
  .leaf-node-content.selected {
    border-color: #ff6b35;
    box-shadow: 0 0 8px rgba(255, 107, 53, 0.3);
  }
  
  /* removed required styling to prevent red nodes */
  
  .leaf-content {
    display: flex;
    flex-direction: column;
    gap: 2px;
  }
  
  .leaf-label {
    font-weight: bold;
    color: #333;
    font-size: 0.85rem;
    line-height: 1.2;
  }
  
  .leaf-type {
    font-size: 0.7rem;
    color: #666;
    background: #f0f0f0;
    padding: 2px 4px;
    border-radius: 2px;
  }
  
  .leaf-path {
    font-size: 0.65rem;
    color: #888;
    font-style: italic;
  }
</style>