<script lang="ts">
  export let selectedNode: any;
  export let onEdit: () => void;
</script>

<div class="overview">
  <h3>Overview</h3>
  
  {#if selectedNode}
    <div class="overview-content">
      <div class="node-info">
        <div class="info-row">
          <strong>Component:</strong> 
          <span>{selectedNode.data?.componentName || 'Unknown'}</span>
        </div>
        
        {#if selectedNode.data?.modeName}
          <div class="info-row">
            <strong>Mode:</strong> 
            <span>{selectedNode.data.modeName}</span>
          </div>
        {/if}
        
        {#if selectedNode.data?.componentId}
          <div class="info-row">
            <strong>ID:</strong> 
            <span>{selectedNode.data.componentId}</span>
          </div>
        {/if}
        
        <div class="info-row">
          <strong>Type:</strong> 
          <span>{selectedNode.type}</span>
        </div>
      </div>
    </div>
  {/if}
  
  <div class="overview-actions">
    <button class="edit-button" on:click={onEdit}>
      Bearbeiten
    </button>
  </div>
</div>

<style>
  .overview {
    padding: 1rem;
    height: 100%;
    display: flex;
    flex-direction: column;
  }
  
  .overview h3 {
    margin: 0 0 1rem 0;
    font-size: 1.2rem;
    color: #333;
  }
  
  .overview-content {
    flex: 1;
    margin-bottom: 1rem;
  }
  
  .node-info {
    background: #f8f9fa;
    border-radius: 6px;
    padding: 1rem;
  }
  
  .info-row {
    margin-bottom: 0.5rem;
    font-size: 0.9rem;
  }
  
  .info-row:last-child {
    margin-bottom: 0;
  }
  
  .info-row strong {
    display: inline-block;
    width: 80px;
    color: #555;
  }
  
  .overview-actions {
    border-top: 1px solid #ddd;
    padding-top: 1rem;
  }
  
  .edit-button {
    width: 100%;
    padding: 0.75rem;
    background: #007acc;
    color: white;
    border: none;
    border-radius: 6px;
    cursor: pointer;
    font-size: 1rem;
    transition: background-color 0.2s;
  }
  
  .edit-button:hover {
    background: #005fa3;
  }
</style>