<script lang="ts">
  import { writable } from 'svelte/store';

  export let componentConfig: any;
  export let validationStatus: any;
  export let edges: any;
  export let selectedMode: any;
  export let componentManifest: any;
  export let selectedNode: any;

  let selectedParameter = '';
  let regexInput = '';
  let showResetConfirm = false;

  // store initial state for reset functionality
  let initialConfigState: any = null;

  // initialize initial state on first load
  $: if (componentConfig && !initialConfigState) {
    initialConfigState = JSON.parse(JSON.stringify(componentConfig));
  }

  $: allVariables = selectedMode?.variables?.variable || [];
  $: allSettings = selectedMode?.settings?.setting || [];
  $: globalSettings = componentManifest?.globalSettings?.globalsetting || [];

  // get current config settings
  $: configSettings = componentConfig?.components?.[0]?.mode?.settings?.setting || [];
  $: configGlobalSettings = componentConfig?.components?.[0]?.globalSettings?.globalsetting || [];

  // create default values lookup from manifest
  $: settingDefaults = createSettingDefaults();
  $: globalDefaults = createGlobalDefaults();

  function createSettingDefaults() {
    const defaults = {};
    if (selectedMode?.settings?.setting) {
      selectedMode.settings.setting.forEach(setting => {
        if (setting.default_value) {
          defaults[setting.target_variable] = setting.default_value.value;
        }
      });
    }
    return defaults;
  }

  function createGlobalDefaults() {
    const defaults = {};
    if (componentManifest?.globalSettings?.globalsetting) {
      componentManifest.globalSettings.globalsetting.forEach(setting => {
        if (setting.default_value) {
          defaults[setting.target_variable] = setting.default_value.value;
        }
      });
    }
    return defaults;
  }

  // initialize settings with default values if not set
  $: {
    if (configSettings.length > 0) {
      configSettings.forEach(setting => {
        if (setting.value === undefined && settingDefaults[setting.target_variable]) {
          setting.value = settingDefaults[setting.target_variable];
        }
      });
    }
  }

  $: if ($edges) {
    console.log('edges changed');
  }

  function applyValidation() {
    if (!selectedParameter || !regexInput) return;
    selectedParameter = '';
    regexInput = '';
  }

  function handleBooleanChange(setting, event) {
    setting.value = event.target.checked;

    componentConfig = componentConfig;
  }

  function handleInputChange(setting, event) {
    setting.value = event.target.value;

    componentConfig = componentConfig;
  }

  function resetToDefaults() {
    showResetConfirm = true;
  }

  function confirmReset() {
    if (initialConfigState) {
      // reset by overwriting current config with initial state
      componentConfig.components = JSON.parse(JSON.stringify(initialConfigState.components));
      
      // force reactivity update
      componentConfig = componentConfig;
      
      // clear validation form fields
      selectedParameter = '';
      regexInput = '';
    }
    showResetConfirm = false;
  }

  function cancelReset() {
    showResetConfirm = false;
  }

  // get setting type from manifest
  function getSettingType(targetVariable) {
    const manifestSetting = selectedMode?.settings?.setting?.find(s => s.target_variable === targetVariable);
    return manifestSetting?.type || 'string';
  }

  function getSettingDefaultValue(targetVariable) {
    const manifestSetting = selectedMode?.settings?.setting?.find(s => s.target_variable === targetVariable);
    if (manifestSetting?.default_value) {
      if (manifestSetting.type === 'boolean') {
        return manifestSetting.default_value.value === 'true';
      }
      return manifestSetting.default_value.value;
    }
    return manifestSetting?.type === 'boolean' ? false : '';
  }

  // get validation status for elected node
  $: nodeValidationStatus = getNodeValidationStatus();

  function getNodeValidationStatus() {
    if (!selectedNode || !$validationStatus.connected.items) {
      return { connected: 0, total: 0, items: {} };
    }

    const nodeId = selectedNode.id;
    const nodeVariables = selectedNode.data?.componentVariables || [];
    
    let connectedCount = 0;
    let nodeItems = {};
    
    nodeVariables.forEach((variable: any) => {
      const itemKey = `${nodeId}-${variable.target_variable}`;
      const isConnected = $validationStatus.connected.items[itemKey] || false;
      nodeItems[variable.target_variable] = isConnected;
      if (isConnected) {
        connectedCount++;
      }
    });
    
    return {
      connected: connectedCount,
      total: nodeVariables.length,
      items: nodeItems
    };
  }
</script>

<div class="config-tab">
  <h4>Configuration</h4>

  <!-- validation summary -->
  <div class="section">
    <h5>Validation Summary</h5>
    <div class="validation-summary" class:valid={nodeValidationStatus.connected === nodeValidationStatus.total && nodeValidationStatus.total > 0}>
      <div class="summary-stats">
        <span class="stat">Connected: {nodeValidationStatus.connected}/{nodeValidationStatus.total}</span>
        <span class="status-badge" class:valid={nodeValidationStatus.connected === nodeValidationStatus.total && nodeValidationStatus.total > 0}>
          {nodeValidationStatus.connected === nodeValidationStatus.total && nodeValidationStatus.total > 0 ? 'All Connected' : 'Missing Connections'}
        </span>
      </div>
    </div>
  </div>

  <!-- reset button -->
  <div class="section">
    <button class="reset-button" on:click={resetToDefaults}>
      Back to Default
    </button>
  </div>

  <!-- settings list -->
  {#if configSettings.length > 0}
    <div class="section">
      <h5>Settings</h5>
      <div class="settings-list">
        {#each configSettings as setting}
          {@const settingType = getSettingType(setting.target_variable)}
          {@const defaultValue = getSettingDefaultValue(setting.target_variable)}
          
          <div class="setting-item" class:boolean={settingType === 'boolean'}>
            {#if settingType === 'boolean'}
              <input 
                type="checkbox" 
                id={`setting-${setting.target_variable}`}
                checked={setting.value === true || setting.value === 'true'}
                on:change={(e) => handleBooleanChange(setting, e)}
              />
              <label for={`setting-${setting.target_variable}`} class="setting-name">
                {setting.name || setting.target_variable}
              </label>
              <div class="default-info">Default: {defaultValue}</div>
            {:else}
              <label class="setting-name">{setting.name || setting.target_variable}:</label>
              <input 
                type="text" 
                value={setting.value || ''} 
                on:input={(e) => handleInputChange(setting, e)}
                placeholder={`Default: ${defaultValue || 'No default'}`}
              />
              <div class="default-info">
                Default: {defaultValue || 'No default'}
              </div>
            {/if}
          </div>
        {/each}
      </div>
    </div>
  {/if}

  <!-- variables validation -->
  <div class="section">
    <h5>Variables Validation</h5>
    <div class="parameters-list">
      {#each allVariables as variable}
        {@const isConnected = nodeValidationStatus.items[variable.target_variable] || false}
        <div class="parameter-item" class:connected={isConnected} class:type-error={!$validationStatus.typeValid}>
          <div class="parameter-info">
            <div class="parameter-name">{variable.target_variable}</div>
            <div class="parameter-details">
              <span class="parameter-type">Type: {variable.type}</span>
              <span class="parameter-direction" class:input={variable.is_input} class:output={variable.is_output}>
                {variable.is_input && variable.is_output ? 'IN/OUT' : variable.is_input ? 'IN' : 'OUT'}
              </span>
            </div>
          </div>
          <div class="parameter-status">
            <div class="connection-status" class:connected={isConnected}>
              {isConnected ? '✓ Connected' : '✗ Not Connected'}
            </div>
            {#if !$validationStatus.typeValid}
              <div class="type-error">Type mismatch!</div>
            {/if}
            {#if variable.validation?.regex}
              <div class="validation-info">
                Regex: <code>{variable.validation.regex}</code>
              </div>
            {/if}
          </div>
        </div>
      {/each}
      
      {#if allVariables.length === 0}
        <div class="no-variables">
          No variables defined for this mode.
        </div>
      {/if}
    </div>
  </div>

  <!-- regex -->
  <div class="section">
    <h5>Regex</h5>
    <div class="validation-form">
      <div class="form-group">
        <label for="parameter-select">Variable:</label>
        <select id="parameter-select" bind:value={selectedParameter}>
          <option value="">Select Variable</option>
          {#each allVariables as variable}
            <option value={variable.target_variable}>{variable.target_variable}</option>
          {/each}
        </select>
      </div>
      <div class="form-group">
        <label for="regex-input">Regex Pattern:</label>
        <input 
          id="regex-input" 
          type="text" 
          bind:value={regexInput} 
          placeholder="Enter regex pattern"
        />
      </div>
      <button 
        class="apply-button" 
        on:click={applyValidation}
        disabled={!selectedParameter || !regexInput}
      >
        Apply Validation
      </button>
    </div>
  </div>
</div>

<!-- reset confirmation modal -->
{#if showResetConfirm}
  <div class="modal-overlay">
    <div class="modal">
      <h3>Reset to Defaults</h3>
      <p>Are you sure you want to reset all settings to their default values? This action cannot be undone.</p>
      <div class="modal-buttons">
        <button class="cancel-button" on:click={cancelReset}>Cancel</button>
        <button class="confirm-button" on:click={confirmReset}>Yes, Reset</button>
      </div>
    </div>
  </div>
{/if}

<style>
  .config-tab {
    padding: 1rem;
    overflow-y: auto;
    height: 100%;
  }
  .config-tab h4 {
    margin: 0 0 1rem 0;
    color: #333;
  }
  .section {
    margin-bottom: 1.5rem;
    padding-bottom: 1rem;
    border-bottom: 1px solid #eee;
  }
  .section:last-child {
    border-bottom: none;
  }
  .section h5 {
    margin: 0 0 0.75rem 0;
    color: #555;
    font-size: 0.9rem;
    text-transform: uppercase;
    letter-spacing: 0.5px;
  }
  .reset-button {
    width: 100%;
    padding: 0.75rem;
    background: #ff742a;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 0.9rem;
    transition: background-color 0.2s;
  }
  .reset-button:hover {
    background: #fa550f;
  }
  .settings-list,
  .parameters-list {
    display: flex;
    flex-direction: column;
    gap: 0.75rem;
  }
  .setting-item {
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    background: white;
    border: 1px solid #ddd;
    border-radius: 6px;
    padding: 0.75rem;
  }
  .setting-item.boolean {
    flex-direction: row;
    align-items: center;
    gap: 0.75rem;
  }
  .setting-item.boolean input[type="checkbox"] {
    margin: 0;
  }
  .setting-item.boolean label {
    margin: 0;
    cursor: pointer;
    flex: 1;
  }
  .setting-item.boolean .default-info {
    flex: none;
  }
  .setting-name {
    font-weight: bold;
    color: #333;
    margin: 0;
  }
  .setting-item input[type="text"] {
    padding: 0.5rem;
    border: 1px solid #ddd;
    border-radius: 4px;
  }
  .default-info {
    font-size: 0.8rem;
    color: #666;
    font-style: italic;
    margin-top: 0.25rem;
  }
  .parameter-item {
    border: 1px solid #ddd;
    border-radius: 6px;
    padding: 0.75rem;
    background: white;
  }
  .parameter-item.connected {
    background: #f8fff8;
    border-color: #28a745;
  }
  .parameter-item.type-error {
    background: #fff5f5;
    border-color: #dc3545;
  }
  .parameter-name {
    font-weight: bold;
    margin-bottom: 0.25rem;
    color: #333;
  }
  .parameter-details {
    display: flex;
    gap: 1rem;
    margin-bottom: 0.5rem;
    flex-wrap: wrap;
  }
  .parameter-type {
    font-size: 0.8rem;
    color: #666;
    background: #f0f0f0;
    padding: 2px 6px;
    border-radius: 3px;
  }
  .parameter-direction {
    font-size: 0.8rem;
    color: #666;
    background: #f0f0f0;
    padding: 2px 6px;
    border-radius: 3px;
  }
  .parameter-direction.input {
    background: #e3f2fd;
    color: #1976d2;
  }
  .parameter-direction.output {
    background: #f3e5f5;
    color: #7b1fa2;
  }
  .connection-status {
    font-size: 0.8rem;
    font-weight: bold;
  }
  .connection-status.connected {
    color: #28a745;
  }
  .connection-status:not(.connected) {
    color: #dc3545;
  }
  .type-error {
    color: #dc3545;
    font-size: 0.8rem;
    font-weight: bold;
    margin-top: 0.25rem;
  }
  .validation-info {
    font-size: 0.75rem;
    color: #666;
    margin-top: 0.25rem;
  }
  .validation-info code {
    background: #f8f9fa;
    padding: 2px 4px;
    border-radius: 2px;
    font-family: monospace;
  }
  .validation-summary {
    background: #f8f9fa;
    border: 1px solid #ddd;
    border-radius: 6px;
    padding: 1rem;
  }
  .validation-summary.valid {
    background: #d4edda;
    border-color: #c3e6cb;
  }
  .summary-stats {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 1rem;
  }
  .stat {
    font-size: 0.9rem;
    font-weight: bold;
    color: #666;
  }
  .validation-summary.valid .stat {
    color: #155724;
  }
  .status-badge {
    padding: 0.25rem 0.75rem;
    border-radius: 20px;
    font-size: 0.8rem;
    font-weight: bold;
    background: #6c757d;
    color: white;
  }
  .status-badge.valid {
    background: #28a745;
    color: white;
  }
  .no-variables {
    text-align: center;
    color: #999;
    font-style: italic;
    padding: 2rem;
  }
  .validation-form {
    background: #f8f9fa;
    padding: 1rem;
    border-radius: 6px;
  }
  .form-group {
    margin-bottom: 1rem;
  }
  .form-group:last-child {
    margin-bottom: 0;
  }
  .form-group label {
    display: block;
    margin-bottom: 0.25rem;
    font-weight: bold;
    color: #555;
    font-size: 0.9rem;
  }
  .form-group select,
  .form-group input {
    width: 100%;
    padding: 0.5rem;
    border: 1px solid #ddd;
    border-radius: 4px;
    font-size: 0.9rem;
  }
  .form-group select:focus,
  .form-group input:focus {
    outline: none;
    border-color: #007acc;
    box-shadow: 0 0 0 2px rgba(0, 122, 204, 0.2);
  }
  .apply-button {
    width: 100%;
    padding: 0.75rem;
    background: #007acc;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 0.9rem;
    margin-top: 0.5rem;
    transition: background-color 0.2s;
  }
  .apply-button:hover:not(:disabled) {
    background: #005fa3;
  }
  .apply-button:disabled {
    background: #ccc;
    cursor: not-allowed;
  }

  /* modal styles */
  .modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 10000;
  }
  .modal {
    background: white;
    border-radius: 8px;
    padding: 2rem;
    max-width: 400px;
    width: 90%;
    box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
  }
  .modal h3 {
    margin: 0 0 1rem 0;
    color: #333;
  }
  .modal p {
    margin: 0 0 1.5rem 0;
    color: #666;
    line-height: 1.5;
  }
  .modal-buttons {
    display: flex;
    gap: 1rem;
    justify-content: flex-end;
  }
  .cancel-button {
    padding: 0.75rem 1.5rem;
    background: #6c757d;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    transition: background-color 0.2s;
  }
  .cancel-button:hover {
    background: #5a6268;
  }
  .confirm-button {
    padding: 0.75rem 1.5rem;
    background: #ec3f0f;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    transition: background-color 0.2s;
  }
  .confirm-button:hover {
    background: #c82333;
  }
</style>