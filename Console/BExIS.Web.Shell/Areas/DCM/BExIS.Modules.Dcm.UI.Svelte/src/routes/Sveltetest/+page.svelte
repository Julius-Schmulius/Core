<script lang="ts">
  import {
    SvelteFlow,
    Background,
    Controls,
    Position,
    SelectionMode,
    ConnectionLineType,
    MarkerType,
    type Node,
    type Edge,
    type EdgeTypes,
    type NodeTypes
  } from '@xyflow/svelte';
  import { writable, type Writable, get } from 'svelte/store';
  import '@xyflow/svelte/dist/style.css';
  import { onMount } from 'svelte';

  // import schema tree component
  import TreeComponent from './TreeComponent.svelte';

  // import custom components
  import NodeWithItems from './NodeWithItems.svelte';
  import ItemNode from './ItemNode.svelte';
  import ButtonEdge from './ButtonEdge.svelte';
  import ComponentOverview from './ComponentOverview.svelte';
  import ConfigTab from './ConfigTab.svelte';
  import PreviewTab from './PreviewTab.svelte';
  import ModesTab from './ModesTab.svelte';
  import ComponentLibrary from './ComponentLibrary.svelte';
  import SectionNode from './SectionNode.svelte';
  import LeafNode from './LeafNode.svelte';

  import componentConfigJson from './componentConfig.json';
  import componentManifestJson from './componentManifest.json';

  // component data sources
  let componentConfig = componentConfigJson;
  const componentManifest = componentManifestJson;

  // schema nodes from treecomponent
  let schemaNodes: any[] = [];

  // global interaction mode from top-bar controls
  let currentInteractionMode = 'edit';
  let selectedMode: any = null;

  // store for node specific modes
  let nodeSpecificModes = new Map<string, any>();

  // popup state for mode switch warning
  let showModeChangeWarning = false;
  let pendingInteractionMode = '';

  // init counter for forcing node updates
  let nodeVersion = 0;

  // reference to svelte flow component for viewport calculations
  let flowElement: any = null;

  // initial selected mode based on first component in config
  onMount(() => {
    // TODO: set initial interaction mode from config
    const initialMode = getInteractionModeFromConfig();
    if (initialMode) {
      currentInteractionMode = initialMode;
    }
    
    const firstComponent = componentConfig?.components?.[0];
    if (firstComponent?.mode?.mode_name) {
      const manifestModes = componentManifest?.modes?.[currentInteractionMode] || [];
      const foundMode = manifestModes.find((mode: any) => mode.mode_name === firstComponent.mode.mode_name);
      if (foundMode) {
        selectedMode = foundMode;
        console.log('initial selectedmode set from config:', selectedMode);
        forceNodeUpdate();
      }
    }
  });

  function getInteractionModeFromConfig(): string {
    return componentConfig?.components?.[0]?.globalSettings?.interaction_mode || 'edit';
  }

  function updateInteractionModeInConfig(newMode: string) {
    console.log('triggering display filter for mode:', newMode);
    forceNodeUpdate();
  }

  function handleInteractionModeToggle(newMode: string) {
    if (newMode === currentInteractionMode) return;
    
    if (sidebarMode === 'edit') {
      pendingInteractionMode = newMode;
      showModeChangeWarning = true;
      return;
    }
    
    applyInteractionModeChange(newMode);
  }

  function applyInteractionModeChange(newMode: string) {
    console.log('changing interaction mode from', currentInteractionMode, 'to', newMode);
    
    currentInteractionMode = newMode;
    updateInteractionModeInConfig(newMode);
    
    if (sidebarMode === 'edit') {
      sidebarMode = 'empty';
      selectedNode.set(null);
    }
    
    const newModes = componentManifest?.modes?.[newMode];
    if (newModes && newModes.length > 0) {
      selectedMode = newModes[0];
      console.log('selectedMode updated == first mode of new interaction mode:', selectedMode);
    } else {
      selectedMode = null;
      console.log('no modes available for interaction mode:', newMode);
    }
    
    forceNodeUpdate();

    setTimeout(() => {
      filterEdgesForInteractionMode();
    }, 100);
  }

  function confirmModeChange() {
    applyInteractionModeChange(pendingInteractionMode);
    showModeChangeWarning = false;
    pendingInteractionMode = '';
  }

  function cancelModeChange() {
    showModeChangeWarning = false;
    pendingInteractionMode = '';
  }

  // save/cancel handlers for edit buttons
  function handleSaveEdit() {
    console.log('save button clicked');
    // TODO: implement save functionality
  }

  function handleCancelEdit() {
    console.log('cancel button clicked');
    // TODO: implement cancel functionality
  }

  $: componentVariables = selectedMode?.variables?.variable || [];
  $: componentSettings = selectedMode?.settings?.setting || [];

  $: {
    console.log('=== reactive update ===');
    console.log('selectedmode:', selectedMode?.mode_name);
    console.log('componentvariables:', componentVariables);
    console.log('componentvariables length:', componentVariables.length);
    if (componentVariables.length > 0) {
      console.log('variables details:', componentVariables.map((v: any) => ({ 
        target: v.target_variable, 
        input: v.is_input, 
        output: v.is_output 
      })));
    }
  }

  const childWidth = 140;
  const childHeight = 60;
  const colGap = 20;
  const rowGap = 12;
  const columns = 1;

  $: rows = Array.isArray(componentVariables) ? componentVariables.length : 0;
  $: parentWidth = Math.max(320, childWidth + 80);
  $: parentHeight = Math.max(140, rows * childHeight + (rows - 1) * rowGap + 100);

  function forceNodeUpdate() {
    nodeVersion++;
    console.log('force node update, version:', nodeVersion);
  }

  // handle schema nodes generation from treecomponent
  function handleSchemaNodesGenerated(event: any) {
    const generatedNodes = event.detail.nodes || [];
    console.log('received schema nodes from treecomponent:', generatedNodes.length);
    schemaNodes = generatedNodes;
  }

  // dynamically create nodes
  $: configNodes = createConfigNodes(currentInteractionMode, componentConfig, componentManifest, nodeVersion);

  function createConfigNodes(interactionMode?: string, config?: any, manifest?: any, version?: number): Node[] {
    const components = config?.components || componentConfig?.components;
    if (!components || !Array.isArray(components)) return [];
    
    const currentMode = interactionMode || currentInteractionMode;
    const currentManifest = manifest || componentManifest;
    
    const nodes: Node[] = [];
    
    components.forEach((component: any, index: number) => {
      console.log('checking component with interaction_mode:', component.globalSettings?.interaction_mode, 'current mode:', currentMode);
      if (component.globalSettings?.interaction_mode === currentMode) {
        const manifestModes = currentManifest?.modes?.[currentMode] || [];
        const modeDetails = manifestModes.find((mode: any) => mode.mode_name === component.mode?.mode_name);
        
        if (modeDetails) {
          const modeVariables = modeDetails.variables?.variable || [];
          
          const childItems = modeVariables.map((variable: any) => ({
            id: variable.target_variable,
            label: variable.target_variable,
            isInput: variable.is_input,
            isOutput: variable.is_output,
            type: variable.type
          }));

          const rows = Array.isArray(modeVariables) ? modeVariables.length : 0;
          const nodeWidth = Math.max(320, childWidth + 80);
          const nodeHeight = Math.max(140, rows * childHeight + (rows - 1) * rowGap + 100);

          const node = {
            id: `config-component-${index}`,
            type: 'nodeWithItems',
            data: {
              label: modeDetails.mode_name || component.meta.component_name,
              componentId: component.meta.component_name,
              componentName: component.meta.component_name,
              modeName: modeDetails.mode_name || '',
              interactionMode: currentMode,
              childItems: childItems,
              componentVariables: modeVariables,
              edges: [],
              version: version || nodeVersion
            },
            position: { x: 400 + (index * 200), y: 100 },
            style: `width: ${nodeWidth}px; height: ${nodeHeight}px;`,
            selectable: true,
            deletable: false,
            selected: false,
            zIndex: 0,
            dragging: false,
            draggable: true
          };
          
          nodes.push(node);
          console.log('added config node:', node.id, 'for mode:', currentMode);
        }
      } else {
        console.log('skipping component - mode mismatch:', component.globalSettings?.interaction_mode, '!==', currentMode);
      }
    });
    
    console.log('created', nodes.length, 'config nodes for mode:', currentMode);
    return nodes;
  }

  function filterEdgesForInteractionMode() {
    const currentEdges = get(edges);
    const currentNodes = get(nodes);
    
    // console.log('=== filtering edges for interaction mode ===');
    // console.log('current mode:', currentInteractionMode);
    // console.log('edges before filter:', currentEdges.length);
    
    const filteredEdges = currentEdges.filter(edge => {

      const sourceNode = currentNodes.find(n => n.id === edge.source);
      const targetNode = currentNodes.find(n => n.id === edge.target);
      const sourceIsComponent = sourceNode?.id?.startsWith('config-component-') || sourceNode?.id?.startsWith('library-component-');
      const targetIsComponent = targetNode?.id?.startsWith('config-component-') || targetNode?.id?.startsWith('library-component-');

      if (sourceIsComponent) {
        const nodeInteractionMode = sourceNode?.data?.interactionMode;
        if (nodeInteractionMode && nodeInteractionMode !== currentInteractionMode) {
          return false;
        }
      }

      if (targetIsComponent) {
        const nodeInteractionMode = targetNode?.data?.interactionMode;
        if (nodeInteractionMode && nodeInteractionMode !== currentInteractionMode) {
          return false;
        }
      }
      
      return true;
    });
    
    // console.log('edges after filter:', filteredEdges.length);
    edges.set(filteredEdges);
  }

  let libraryNodes: Writable<Node[]> = writable([]);

  $: filteredLibraryNodes = $libraryNodes.filter(node => {
    return node.data?.interactionMode === currentInteractionMode;
  });

  // merge all nodes & replace parameter nodes with schema nodes
  $: {
    const allNodes: Node[] = [];
    
    if (configNodes && Array.isArray(configNodes) && configNodes.length > 0) {
      const updatedConfigNodes = configNodes.map(node => ({
        ...node,
        data: {
          ...node.data,
          edges: $edges || [],
          version: nodeVersion
        }
      }));
      allNodes.push(...updatedConfigNodes);
    }
    
    // replace parameter nodes with schema nodes
    if (schemaNodes && Array.isArray(schemaNodes) && schemaNodes.length > 0) {
      allNodes.push(...schemaNodes);
    }
    
    if (filteredLibraryNodes && Array.isArray(filteredLibraryNodes)) {
      const updatedLibraryNodes = filteredLibraryNodes.map(node => ({
        ...node,
        data: {
          ...node.data,
          edges: $edges || [],
          version: nodeVersion
        }
      }));
      allNodes.push(...updatedLibraryNodes);
    }
    
    console.log('setting nodes. config nodes count:', configNodes.length);
    console.log('schema nodes count:', schemaNodes.length);
    console.log('filtered library nodes count:', filteredLibraryNodes.length);
    
    nodes.set(allNodes);
  }

  let nodes: Writable<Node[]> = writable([]);
  let edges: Writable<Edge[]> = writable([]);

  let selectedNode: Writable<Node | null> = writable(null);
  let selectedEdge: Writable<Edge | null> = writable(null);

  let sidebarMode: 'empty' | 'overview' | 'edit' = 'empty';
  let activeTab = 0;

  let validationStatus = writable({
    connected: {
      total: 0,
      connected: 0,
      items: {}
    },
    isValid: false,
    typeValid: true
  });

  // update validation status based on selected node
  $: {
    if ($selectedNode && Array.isArray(componentVariables) && componentVariables.length > 0) {
      const selectedNodeVariables = $selectedNode.data?.componentVariables || [];
      validationStatus.update(status => ({
        ...status,
        connected: {
          ...status.connected,
          total: Array.isArray(selectedNodeVariables) ? selectedNodeVariables.length : 0
        }
      }));
      setTimeout(validateConnections, 10);
    }
  }

  $: currentNodeMode = getCurrentNodeMode();

  function getCurrentNodeMode() {
    if (!$selectedNode) return selectedMode; // default if none
    
    if (nodeSpecificModes.has($selectedNode.id)) {
      return nodeSpecificModes.get($selectedNode.id);
    }
    
    if ($selectedNode.data?.modeName) {
      const manifestModes = componentManifest?.modes?.[currentInteractionMode] || [];
      const nodeMode = manifestModes.find((mode: any) => mode.mode_name === $selectedNode.data.modeName);
      if (nodeMode) {
        nodeSpecificModes.set($selectedNode.id, nodeMode);
        return nodeMode;
      }
    }
    
    return selectedMode;
  }

  function handleModeChange(newMode: any) {
    console.log('=== component mode change ===');
    console.log('changing to:', newMode.mode_name);
    console.log('new mode variables:', newMode.variables?.variable);
    
    if (!$selectedNode) {
      console.log('no selected node for mode change');
      return;
    }

    // store current position before update
    const currentPosition = $selectedNode.position;
    console.log('preserving position:', currentPosition);

    nodeSpecificModes.set($selectedNode.id, newMode);
    console.log('storing mode for node:', $selectedNode.id, newMode.mode_name);

    const newVariables = newMode.variables?.variable || [];
    const newChildItems = newVariables.map((variable: any) => ({
      id: variable.target_variable,
      label: variable.target_variable,
      isInput: variable.is_input,
      isOutput: variable.is_output,
      type: variable.type
    }));

    // dynamic sizing for mode change
    const rows = newVariables.length;
    const newWidth = Math.max(320, childWidth + 80);
    const newHeight = Math.max(140, rows * childHeight + (rows - 1) * rowGap + 100);
    
    console.log('updating node with new childItems:', newChildItems);

    nodes.update(allNodes => allNodes.map(node => {
      if (node.id === $selectedNode.id) {
        return {
          ...node,
          position: currentPosition, // explicitly preserve position
          data: {
            ...node.data,
            modeName: newMode.mode_name,
            childItems: newChildItems,
            componentVariables: newVariables,
            version: nodeVersion + 1
          },
          style: `width: ${newWidth}px; height: ${newHeight}px;`
        };
      }
      return node;
    }));

    // update dynamic library nodes if necessary
    if ($selectedNode.id.startsWith('library-component-')) {
      libraryNodes.update(libNodes => libNodes.map(node => {
        if (node.id === $selectedNode.id) {
          return {
            ...node,
            position: currentPosition, // also preserve here
            data: {
              ...node.data,
              modeName: newMode.mode_name,
              childItems: newChildItems,
              componentVariables: newVariables,
              version: nodeVersion + 1
            },
            style: `width: ${newWidth}px; height: ${newHeight}px;`
          };
        }
        return node;
      }));
    }

    // filter dynamic edges & remove edges belonging to changed handles
    edges.update(allEdges => allEdges.filter(edge => 
      !edge.sourceHandle?.startsWith($selectedNode.id + '-') &&
      !edge.targetHandle?.startsWith($selectedNode.id + '-')
    ));
    
    selectedMode = newMode;
    componentVariables = newMode.variables?.variable || [];
    componentSettings = newMode.settings?.setting || [];
    
    console.log('after mode change - componentvariables:', componentVariables);
    console.log('after mode change - variables length:', componentVariables.length);
    
    // if forceNodeUpdate() cant be called recreate all nodes
    // forceNodeUpdate();
  }

  function getViewportCenter(): {x: number, y: number} {
    if (flowElement && flowElement.getViewport) {
      const viewport = flowElement.getViewport();
      const flowContainer = flowElement.domNode || flowElement;
      const flowRect = flowContainer.getBoundingClientRect?.() || { width: 800, height: 600 };
      
      console.log('viewport info:', viewport);
      console.log('flow rect:', flowRect);
      
      const centerX = (-viewport.x + flowRect.width / 2) / viewport.zoom;
      const centerY = (-viewport.y + flowRect.height / 2) / viewport.zoom;
      
      console.log('calculated center:', { x: centerX, y: centerY });
      
      return { x: centerX - 160, y: centerY - 100 };
    }
    
    console.log('using fallback center position');
    return { x: 400, y: 300 };
  }

  function handleAddComponent(component: any) {
    console.log('adding component to flow:', component);
    
    const centerPos = getViewportCenter();
    console.log('center position for new component:', centerPos);
    
    const modeVariables = component.mode?.variables?.variable || [];
    
    // create dynamic child items for new componentg
    const childItems = modeVariables.map((variable: any) => ({
      id: variable.target_variable,
      label: variable.target_variable,
      isInput: variable.is_input,
      isOutput: variable.is_output,
      type: variable.type
    }));

    // dynamic sizing for new component
    const rows = Array.isArray(modeVariables) ? modeVariables.length : 0;
    const newParentWidth = Math.max(320, childWidth + 80);
    const newParentHeight = Math.max(140, rows * childHeight + (rows - 1) * rowGap + 100);

    const newNodeId = `library-component-${Date.now()}`;
    const newNode = {
      id: newNodeId,
      type: 'nodeWithItems',
      data: {
        label: component.meta.title || component.meta.component_name,
        componentId: component.meta.component_name,
        componentName: component.meta.component_name,
        modeName: component.mode?.mode_name || '',
        interactionMode: currentInteractionMode,
        childItems: childItems,
        componentVariables: modeVariables,
        edges: [],
        version: 0
      },
      position: centerPos,
      style: `width: ${newParentWidth}px; height: ${newParentHeight}px;`,
      selectable: true,
      deletable: true,
      selected: false,
      zIndex: 0,
      dragging: false,
      draggable: true
    };
    
    libraryNodes.update(current => [...current, newNode]);
    
    console.log('added new node:', newNode);
  }

  // find initial direction
  function determineInitialDirection(sourceHandleId: string, targetHandleId: string) {
    console.log('=== determining initial direction ===');
    console.log('sourceHandleId:', sourceHandleId);
    console.log('targetHandleId:', targetHandleId);
    
    let componentVariablesToCheck: any[] = [];
    let componentHandleId = '';
    
    const allNodes = get(nodes);
    
    // find dynamic component node
    for (const node of allNodes) {
      if (node.type === 'nodeWithItems') {
        if (sourceHandleId && sourceHandleId.startsWith(`${node.id}-`) && sourceHandleId.endsWith('-handle')) {
          componentVariablesToCheck = Array.isArray(node.data?.componentVariables) ? node.data.componentVariables : [];
          componentHandleId = sourceHandleId;
          break;
        } else if (targetHandleId && targetHandleId.startsWith(`${node.id}-`) && targetHandleId.endsWith('-handle')) {
          componentVariablesToCheck = Array.isArray(node.data?.componentVariables) ? node.data.componentVariables : [];
          componentHandleId = targetHandleId;
          break;
        }
      }
    }
    
    console.log('found component variables:', componentVariablesToCheck);

    // check if component handle and variables are valid & set bools for edge-end arrow markers
    if (componentHandleId && componentVariablesToCheck.length > 0) {
      // extract dynamic variable name
      const handleParts = componentHandleId.split('-');
      console.log('handle parts:', handleParts);
      
      if (handleParts.length >= 3) {
        const variableName = handleParts[handleParts.length - 2];
        console.log('extracted variable name (corrected):', variableName);

        // find dynamic variable
        const variable = componentVariablesToCheck.find((v: any) => v.target_variable === variableName);
        console.log('found variable:', variable);
        
        if (variable) {
          // automatic bidirectionality
          if (variable.is_input && variable.is_output) {
            console.log('setting bidirectional for input/output parameter:', variableName);
            return { leftDirection: true, rightDirection: true };
          }
          //   input-only
          if (variable.is_input && !variable.is_output) {
            console.log('setting right direction for input-only parameter:', variableName);
            return { leftDirection: false, rightDirection: true };
          }
          // output-only
          if (!variable.is_input && variable.is_output) {
            console.log('setting left direction for output-only parameter:', variableName);
            return { leftDirection: true, rightDirection: false };
          }
        } else {
          console.log('variable not found for name:', variableName);
        }
      } else {
        console.log('unexpected handle format, parts length:', handleParts.length);
      }
    } else {
      console.log('no component handle found in connection');
    }
    
    console.log('using default direction (right only)');
    return { leftDirection: false, rightDirection: true };
  }

  // checks edges for correct directions, if not -> calculate and set again
  function fixExistingEdges() {
    const currentEdges = get(edges);
    console.log('=== fixing existing edges ===');
    console.log('current edges before fix:', currentEdges);
    
    if (!Array.isArray(currentEdges)) {
      console.log('no valid edges array');
      return;
    }
    
    let hasChanges = false;
    
    const fixedEdges = currentEdges.map(edge => {
      const hasValidDirections = edge.data && 
        (edge.data.leftDirection === true || edge.data.leftDirection === false) &&
        (edge.data.rightDirection === true || edge.data.rightDirection === false);
      
      if (hasValidDirections) {
        console.log('edge already has valid direction data:', edge.id, edge.data);
        return edge;
      }
      
      console.log('fixing edge with missing/invalid direction data:', edge.id, 'current data:', edge.data);
      
      const direction = determineInitialDirection(edge.sourceHandle ?? '', edge.targetHandle ?? '');
      
      console.log('determined direction for edge:', edge.id, direction);
      
      const fixedEdge = {
        ...edge,
        data: {
          ...edge.data,
          leftDirection: direction.leftDirection,
          rightDirection: direction.rightDirection,
          sourceHandleId: edge.sourceHandle,
          targetHandleId: edge.targetHandle
        },

        markerEnd: direction.leftDirection ? { type: MarkerType.ArrowClosed, color: '#007acc' } : undefined,
        markerStart: direction.rightDirection ? { type: MarkerType.ArrowClosed, color: '#007acc' } : undefined
      };
      
      console.log('fixed edge:', fixedEdge);
      hasChanges = true;
      return fixedEdge;
    });
    
    if (hasChanges) {
      console.log('updating edges with fixed data');
      edges.set(fixedEdges);
      
      setTimeout(validateConnections, 100);
    } else {
      console.log('no edges needed fixing');
    }
  }

  $: if (Array.isArray(componentVariables) && componentVariables.length > 0) {
    setTimeout(fixExistingEdges, 50);
  }

  function handleNodeClick(event: any) {
    const node = event.detail.node;
    selectedNode.set(node);
    selectedEdge.set(null);
    
    edges.update(eds => eds.map(edge => ({ ...edge, selected: false })));
    
    if (node.type === 'nodeWithItems' || node.type === 'group') {
      sidebarMode = 'overview';
    }
  }

  function handleEdgeClick(event: any) {
    const edge = event.detail.edge;
    console.log('edge clicked:', edge);
    selectedEdge.set(edge);
    selectedNode.set(null);
    
    edges.update(eds => eds.map(e => ({ ...e, selected: e.id === edge.id })));
  }

  function handlePaneClick() {
    selectedNode.set(null);
    selectedEdge.set(null);
    sidebarMode = 'empty';
    
    edges.update(eds => eds.map(edge => ({ ...edge, selected: false })));
  }

  function handleEdit() {
    sidebarMode = 'edit';
    activeTab = 0;
  }

  // check if connection is valid before allowing
  function isValidConnection(connection: any) {
    if (!connection) return true;
    
    // console.log('=== isvalidconnection check ===');
    // console.log('connection:', connection);
    
    // check input restriction
    if (connection.sourceHandle && connection.sourceHandle.includes('-handle')) {
      const currentEdges = get(edges);
      const currentNodes = get(nodes);
      const initialDirection = determineInitialDirection(connection.sourceHandle, connection.targetHandle);
      
      const sourceNode = currentNodes.find(n => connection.sourceHandle.startsWith(n.id + '-'));
      const isComponentNode = sourceNode?.type === 'nodeWithItems' || sourceNode?.id?.startsWith('config-component-') || sourceNode?.id?.startsWith('library-component-');
      
      // check flow direction for components
      if (isComponentNode && Array.isArray(currentEdges)) {
        const newEdgeWouldHaveInput = initialDirection.rightDirection || (initialDirection.leftDirection && initialDirection.rightDirection);
        
        if (newEdgeWouldHaveInput) {
          const sourceNodeMode = sourceNode?.data?.interactionMode;
          
          // check all existing input connections
          const existingInputConnections = currentEdges.filter(edge => {
            if (edge.sourceHandle !== connection.sourceHandle) return false;
            
            const hasInputFlow = edge.data?.rightDirection === true || (edge.data?.leftDirection === true && edge.data?.rightDirection === true);
            if (!hasInputFlow) return false;
            
            const edgeSourceNode = currentNodes.find(n => edge.sourceHandle?.startsWith(n.id + '-'));
            const edgeSourceNodeMode = edgeSourceNode?.data?.interactionMode;
            
            return edgeSourceNodeMode === sourceNodeMode;
          });
          
          if (existingInputConnections.length > 0) {
            //console.log('target already has input connection, blocking');
            return false;
          }
        }
      }
    }
    
    return true;
  }

  function onConnect(params: any) {
    if (!params) return;
    
    // console.log('=== onconnect called ===');
    // console.log('connection params:', params);
    
    // input restriction validation
    if (params.sourceHandle && params.sourceHandle.includes('-handle')) {
      const currentEdges = get(edges);
      const currentNodes = get(nodes);
      const initialDirection = determineInitialDirection(params.sourceHandle, params.targetHandle);
      const sourceNode = currentNodes.find(n => params.sourceHandle.startsWith(n.id + '-'));
      
      const isComponentNode = sourceNode?.type === 'nodeWithItems' || sourceNode?.id?.startsWith('config-component-') || sourceNode?.id?.startsWith('library-component-');
      // check flow direction for components
      if (isComponentNode && Array.isArray(currentEdges)) {
        const newEdgeWouldHaveInput = initialDirection.rightDirection || (initialDirection.leftDirection && initialDirection.rightDirection);
        
        if (newEdgeWouldHaveInput) {
          const sourceNodeMode = sourceNode?.data?.interactionMode;
          const existingInputConnections = currentEdges.filter(edge => {
            if (edge.sourceHandle !== params.sourceHandle) return false;
            
            const hasInputFlow = edge.data?.rightDirection === true || (edge.data?.leftDirection === true && edge.data?.rightDirection === true);
            if (!hasInputFlow) return false;
            
            const edgeSourceNode = currentNodes.find(n => edge.sourceHandle?.startsWith(n.id + '-'));
            const edgeSourceNodeMode = edgeSourceNode?.data?.interactionMode;
            
            return edgeSourceNodeMode === sourceNodeMode;
          });
          
          if (existingInputConnections.length > 0) {
            //console.log('target already has input connection, blocking new input connection');
            return;
          }
        }
      }
    }
    
    const initialDirection = determineInitialDirection(params.sourceHandle, params.targetHandle);
    const currentNodes = get(nodes);
    const sourceNode = currentNodes.find(n => params.sourceHandle?.startsWith(n.id + '-'));
    const sourceNodeMode = sourceNode?.data?.interactionMode;

    // create new edge + direction markers
    const newEdge = {
      id: `${params.source}-${params.target}`,
      source: params.source,
      target: params.target,
      sourceHandle: params.sourceHandle,
      targetHandle: params.targetHandle,
      type: 'button',
      animated: false,
      style: 'stroke: #007acc; stroke-width: 2px;',
      selected: false,

      markerEnd: initialDirection.leftDirection ? { type: MarkerType.ArrowClosed, color: '#007acc' } : undefined,
      markerStart: initialDirection.rightDirection ? { type: MarkerType.ArrowClosed, color: '#007acc' } : undefined,
      data: {
        leftDirection: initialDirection.leftDirection,
        rightDirection: initialDirection.rightDirection,
        sourceHandleId: params.sourceHandle,
        targetHandleId: params.targetHandle,
        sourceMode: sourceNodeMode
      }
    };
    
    //console.log('creating new edge with markers and data:', newEdge);
    edges.update(all => [...all, newEdge]);
    
    setTimeout(() => {
      validateConnections();
    }, 10);
  }

  function validateConnections() {
    const currentEdges = get(edges);
    const allNodes = get(nodes);
    let totalConnectedCount = 0;
    let totalVariablesCount = 0;
    let allItems: { [key: string]: boolean } = {};
    let typeValid = true;

    console.log('=== validating connections ===');
    console.log('current interaction mode:', currentInteractionMode);
    console.log('sidebar mode:', sidebarMode);

    // validate based on mode and sidebar state
    if ($selectedNode && sidebarMode === 'edit' && $selectedNode.type === 'nodeWithItems') {
      // edit mode: validate only selected component node
      const selectedNodeVariables = Array.isArray($selectedNode.data?.componentVariables) ? $selectedNode.data.componentVariables : [];
      console.log(`validating selected node ${$selectedNode.id} with ${selectedNodeVariables.length} variables`);
      
      if (selectedNodeVariables.length === 0) {
        validationStatus.set({
          connected: {
            total: 0,
            connected: 0,
            items: {}
          },
          isValid: false,
          typeValid
        });
        return;
      }
      
      totalVariablesCount = selectedNodeVariables.length;

      selectedNodeVariables.forEach((variable: any) => {
        const handleId = `${$selectedNode.id}-${variable.target_variable}-handle`;
        
        console.log(`checking variable ${variable.target_variable} in selected node ${$selectedNode.id} (${variable.is_input ? 'IN' : ''}${variable.is_output ? 'OUT' : ''})`);

        const connectedEdges = currentEdges.filter(edge => {
          const isSourceMatch = edge.sourceHandle === handleId;
          const isTargetMatch = edge.targetHandle === handleId;
          return isSourceMatch || isTargetMatch;
        });
        
        console.log(`- found ${connectedEdges.length} connected edges for ${variable.target_variable}`);
        
        let hasValidConnection = false;

        if (variable.is_input && variable.is_output) {
          const bidirectionalEdges = connectedEdges.filter(edge => 
            edge.data?.leftDirection === true && edge.data?.rightDirection === true
          );
          
          if (bidirectionalEdges.length > 0) {
            hasValidConnection = true;
            console.log(`- ${variable.target_variable} has bidirectional connection`);
          } else {
            const inputConnections = connectedEdges.filter(edge => {
              const isIncoming = edge.targetHandle === handleId && (edge.source.startsWith('schema-') || edge.source.startsWith('param-'));
              const hasRightArrow = edge.data?.rightDirection === true && edge.data?.leftDirection === false;
              return isIncoming && hasRightArrow;
            });
            
            const outputConnections = connectedEdges.filter(edge => {
              const isOutgoing = edge.sourceHandle === handleId && (edge.target.startsWith('schema-') || edge.target.startsWith('param-'));
              const hasLeftArrow = edge.data?.leftDirection === true && edge.data?.rightDirection === false;
              return isOutgoing && hasLeftArrow;
            });
            
            if (inputConnections.length > 0 && outputConnections.length > 0) {
              hasValidConnection = true;
              console.log(`- ${variable.target_variable} has separate input (${inputConnections.length}) and output (${outputConnections.length}) connections`);
            }
          }
          
        } else if (variable.is_input && !variable.is_output) {
          const inputConnections = connectedEdges.filter(edge => {
            const isIncoming = edge.targetHandle === handleId && (edge.source.startsWith('schema-') || edge.source.startsWith('param-'));
            const hasRightArrow = edge.data?.rightDirection === true;
            return isIncoming && hasRightArrow;
          });
          
          if (inputConnections.length > 0) {
            hasValidConnection = true;
            console.log(`- ${variable.target_variable} has input connection`);
          } else {
            console.log(`- ${variable.target_variable} missing input connection`);
          }
          
        } else if (!variable.is_input && variable.is_output) {
          const outputConnections = connectedEdges.filter(edge => {
            const isOutgoing = edge.sourceHandle === handleId && (edge.target.startsWith('schema-') || edge.target.startsWith('param-'));
            const hasLeftArrow = edge.data?.leftDirection === true;
            return isOutgoing && hasLeftArrow;
          });
          
          if (outputConnections.length > 0) {
            hasValidConnection = true;
            console.log(`- ${variable.target_variable} has output connection`);
          } else {
            console.log(`- ${variable.target_variable} missing output connection`);
          }
        }
        
        allItems[`${$selectedNode.id}-${variable.target_variable}`] = hasValidConnection;
        if (hasValidConnection) {
          totalConnectedCount++;
        }
        
        console.log(`- final result for ${variable.target_variable}: ${hasValidConnection}`);
      });
      
      console.log(`selected node validation: ${totalConnectedCount}/${totalVariablesCount} connected`);
      
    } else {
      // overview/view mode: validate all component nodes
      allNodes.forEach(node => {
        if (node.type === 'nodeWithItems') {
          const nodeVariables = Array.isArray(node.data?.componentVariables) ? node.data.componentVariables : [];
          console.log(`validating node ${node.id} with ${nodeVariables.length} variables`);
          
          if (nodeVariables.length === 0) return;
          
          let nodeConnectedCount = 0;
          let nodeVariablesCount = nodeVariables.length;

          nodeVariables.forEach((variable: any) => {
            const handleId = `${node.id}-${variable.target_variable}-handle`;
            
            const connectedEdges = currentEdges.filter(edge => {
              const isSourceMatch = edge.sourceHandle === handleId;
              const isTargetMatch = edge.targetHandle === handleId;
              return isSourceMatch || isTargetMatch;
            });
            
            let hasValidConnection = false;
            
            if (variable.is_input && variable.is_output) {
              const bidirectionalEdges = connectedEdges.filter(edge => 
                edge.data?.leftDirection === true && edge.data?.rightDirection === true
              );
              
              if (bidirectionalEdges.length > 0) {
                hasValidConnection = true;
              } else {
                const inputConnections = connectedEdges.filter(edge => {
                  const isIncoming = edge.targetHandle === handleId && (edge.source.startsWith('schema-') || edge.source.startsWith('param-'));
                  const hasRightArrow = edge.data?.rightDirection === true && edge.data?.leftDirection === false;
                  return isIncoming && hasRightArrow;
                });
                
                const outputConnections = connectedEdges.filter(edge => {
                  const isOutgoing = edge.sourceHandle === handleId && (edge.target.startsWith('schema-') || edge.target.startsWith('param-'));
                  const hasLeftArrow = edge.data?.leftDirection === true && edge.data?.rightDirection === false;
                  return isOutgoing && hasLeftArrow;
                });
                
                if (inputConnections.length > 0 && outputConnections.length > 0) {
                  hasValidConnection = true;
                }
              }
              
            } else if (variable.is_input && !variable.is_output) {
              const inputConnections = connectedEdges.filter(edge => {
                const isIncoming = edge.targetHandle === handleId && (edge.source.startsWith('schema-') || edge.source.startsWith('param-'));
                const hasRightArrow = edge.data?.rightDirection === true;
                return isIncoming && hasRightArrow;
              });
              
              if (inputConnections.length > 0) {
                hasValidConnection = true;
              }
              
            } else if (!variable.is_input && variable.is_output) {
              const outputConnections = connectedEdges.filter(edge => {
                const isOutgoing = edge.sourceHandle === handleId && (edge.target.startsWith('schema-') || edge.target.startsWith('param-'));
                const hasLeftArrow = edge.data?.leftDirection === true;
                return isOutgoing && hasLeftArrow;
              });
              
              if (outputConnections.length > 0) {
                hasValidConnection = true;
              }
            }
            
            allItems[`${node.id}-${variable.target_variable}`] = hasValidConnection;
            if (hasValidConnection) {
              nodeConnectedCount++;
            }
          });
          
          totalVariablesCount += nodeVariablesCount;
          totalConnectedCount += nodeConnectedCount;
        }
      });
    }

    console.log('validation results:', {
      total: totalVariablesCount,
      connected: totalConnectedCount,
      items: allItems
    });

    validationStatus.set({
      connected: {
        total: totalVariablesCount,
        connected: totalConnectedCount,
        items: allItems
      },
      isValid: totalConnectedCount === totalVariablesCount,
      typeValid
    });
  }

  $: if ($edges) {
    validateConnections();
  }

  const nodeTypes: NodeTypes = {
    nodeWithItems: NodeWithItems as any,
    itemNode: ItemNode as any,
    sectionNode: SectionNode as any,  // for section headers
    leafNode: LeafNode as any,        // for metadata leaves with handles
    group: NodeWithItems as any
  };

  const edgeTypes: EdgeTypes = {
    button: ButtonEdge as any
  };
</script>

<style>
  .app-container {
    display: flex;
    flex-direction: column;
    height: 100vh;
    width: 100%;
  }
  
  .top-bar {
    height: 60px;
    background: #ffffff;
    border-bottom: 2px solid #007acc;
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0 2rem;
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
    z-index: 1000;
  }
  
  .project-name {
    font-size: 1.2rem;
    font-weight: bold;
    color: #333;
  }
  
  .mode-controls {
    display: flex;
    gap: 0.5rem;
  }
  
  .mode-button {
    padding: 0.5rem 1.5rem;
    border: 2px solid #007acc;
    background: #ffffff;
    color: #007acc;
    border-radius: 6px;
    cursor: pointer;
    font-weight: bold;
    transition: all 0.2s;
  }
  
  .mode-button.active {
    background: #007acc;
    color: white;
  }
  
  .mode-button:hover:not(.active) {
    background: #f0f8ff;
  }
  
  .layout {
    display: flex;
    flex: 1;
    height: calc(100vh - 60px);
    width: 100%;
  }
  
  .flow-wrapper {
    flex: 1;
    position: relative;
  }
  
  .sidebar {
    width: 350px;
    background: #f5f5f5;
    border-left: 1px solid #ddd;
    display: flex;
    flex-direction: column;
  }
  
  .sidebar.empty {
    background: #fafafa;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #999;
    font-style: italic;
  }
  
  .tabs {
    display: flex;
    border-bottom: 1px solid #ddd;
  }
  
  .tab {
    flex: 1;
    padding: 0.5rem;
    background: #e0e0e0;
    border: none;
    cursor: pointer;
    font-size: 0.9rem;
    transition: background-color 0.2s;
  }
  
  .tab.active {
    background: #f5f5f5;
    border-bottom: 2px solid #007acc;
  }
  
  .tab:hover:not(.active) {
    background: #d5d5d5;
  }
  
  .tab-content {
    flex: 1;
    overflow: hidden;
    display: flex;
    flex-direction: column;
    min-height: 0;
  }

  /* save/cancel buttons styles */
  .edit-actions {
    display: flex;
    gap: 0.75rem;
    padding: 1rem;
    border-top: 1px solid #ddd;
    background: #f8f9fa;
    margin-top: auto;
  }
  
  .save-edit-button,
  .cancel-edit-button {
    flex: 1;
    padding: 0.75rem 1rem;
    border: none;
    border-radius: 6px;
    cursor: pointer;
    font-size: 0.9rem;
    font-weight: bold;
    transition: all 0.2s;
  }
  
  .save-edit-button {
    background: #28a745;
    color: white;
  }
  
  .save-edit-button:hover {
    background: #218838;
    transform: translateY(-1px);
    box-shadow: 0 2px 4px rgba(40, 167, 69, 0.3);
  }
  
  .cancel-edit-button {
    background: #6c757d;
    color: white;
  }
  
  .cancel-edit-button:hover {
    background: #5a6268;
    transform: translateY(-1px);
    box-shadow: 0 2px 4px rgba(108, 117, 125, 0.3);
  }

  :global(.svelte-flow__edges) {
    z-index: 10;
  }

  :global(.svelte-flow__nodes) {
    z-index: 5;
  }

  :global(.svelte-flow__edge) {
    pointer-events: all !important;
    cursor: pointer;
  }

  :global(.svelte-flow__edge path) {
    pointer-events: stroke !important;
    stroke-width: 2px !important;
  }

  :global(.svelte-flow__edge.selected path) {
    stroke: #ff6b35 !important;
    stroke-width: 3px !important;
  }

  :global(.svelte-flow__edge-label) {
    z-index: 9999 !important;
    pointer-events: all !important;
  }

  :global(.svelte-flow__handle) {
    z-index: 20 !important;
    width: 8px !important;
    height: 8px !important;
    background: #007acc !important;
    border: 2px solid white !important;
  }

  :global(.svelte-flow__handle:hover) {
    background: #005fa3 !important;
  }
  
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
    max-width: 500px;
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
    background: #dc3545;
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

<div class="app-container">
  <div class="top-bar">
    <div class="project-name">
      Data Configuration Project
    </div>
    
    <div class="mode-controls">
      <button 
        class="mode-button" 
        class:active={currentInteractionMode === 'edit'}
        on:click={() => handleInteractionModeToggle('edit')}
      >
        EDIT
      </button>
      <button 
        class="mode-button" 
        class:active={currentInteractionMode === 'view'}
        on:click={() => handleInteractionModeToggle('view')}
      >
        VIEW
      </button>
    </div>
  </div>

  <div class="layout">
    <div class="flow-wrapper">
      <SvelteFlow
        {nodes}
        {edges}
        {nodeTypes}
        {edgeTypes}
        bind:this={flowElement}
        on:nodeclick={handleNodeClick}
        on:edgeclick={handleEdgeClick}
        on:paneclick={handlePaneClick}
        on:connect={onConnect}
        {isValidConnection}
        defaultEdgeOptions={{type: 'button'}}
        fitView
        selectionMode={SelectionMode.Partial}
        connectionLineType={ConnectionLineType.SmoothStep}
      >
        <Background />
        <Controls />
      </SvelteFlow>
    </div>

    <div class="sidebar" class:empty={sidebarMode === 'empty'}>
      {#if sidebarMode === 'empty'}
        <ComponentLibrary 
          {currentInteractionMode}
          {componentConfig}
          {componentManifest}
          onAddComponent={handleAddComponent}
        />
      {:else if sidebarMode === 'overview'}
        <ComponentOverview 
          selectedNode={$selectedNode} 
          onEdit={handleEdit}
        />
      {:else if sidebarMode === 'edit'}
        <div class="tabs">
          <button class="tab" class:active={activeTab === 0} on:click={() => activeTab = 0}>
            Modes
          </button>
          <button class="tab" class:active={activeTab === 1} on:click={() => activeTab = 1}>
            Config
          </button>
          <button class="tab" class:active={activeTab === 2} on:click={() => activeTab = 2}>
            Preview
          </button>
        </div>
        
        <div class="tab-content">
          {#if activeTab === 0}
            <ModesTab 
              {componentManifest}
              selectedMode={currentNodeMode}
              {currentInteractionMode}
              selectedNode={$selectedNode}
              onModeChange={handleModeChange}
            />
          {:else if activeTab === 1}
            <ConfigTab 
              {componentConfig}
              {validationStatus}
              {edges}
              selectedMode={selectedMode}
              {componentManifest}
              selectedNode={$selectedNode}
            />
          {:else if activeTab === 2}
            <PreviewTab 
              {componentManifest}
              selectedNode={$selectedNode}
              selectedMode={selectedMode}
              {currentInteractionMode}
              {componentConfig}
            />
          {/if}
        </div>
        
        <!-- save/cancel buttons -->
        <div class="edit-actions">
          <button class="cancel-edit-button" on:click={handleCancelEdit}>
            Cancel
          </button>
          <button class="save-edit-button" on:click={handleSaveEdit}>
            Save
          </button>
        </div>
      {/if}
    </div>
  </div>
</div>

{#if showModeChangeWarning}
  <div class="modal-overlay">
    <div class="modal">
      <h3>Switch Interaction Mode</h3>
      <p>You are currently editing a component. Switching to {pendingInteractionMode.toUpperCase()} mode will close the editor and any unsaved changes will be lost.</p>
      <p>Do you want to continue?</p>
      <div class="modal-buttons">
        <button class="cancel-button" on:click={cancelModeChange}>Cancel</button>
        <button class="confirm-button" on:click={confirmModeChange}>Yes, Switch Mode</button>
      </div>
    </div>
  </div>
{/if}

<!-- schema tree component for node generation -->
<TreeComponent on:nodesGenerated={handleSchemaNodesGenerated} />