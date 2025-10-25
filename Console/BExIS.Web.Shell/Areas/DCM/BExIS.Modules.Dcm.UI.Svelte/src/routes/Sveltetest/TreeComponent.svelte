<script lang="ts">
    import * as ApiCalls from './Services/apiCalls';
    import ComplexComponent from './complexComponentWrapper.svelte';
    import { createEventDispatcher } from 'svelte';

    const dispatch = createEventDispatcher();

    let schema: any;
    let s: any;
    let currentSchema: number = 2;
    let generatedNodes: any[] = [];
    
    $: schema = s;

    async function loader(currentSchema: number) {
        try {
            schema = await ApiCalls.GetMetadataSchema(currentSchema);
            s = schema;
            
            // generate nodes after schema is loaded
            if (schema) {
                generatedNodes = generateNodesFromSchema(schema);
                dispatch('nodesGenerated', { nodes: generatedNodes });
            }
        } catch (error) {
            console.error('error loading metadata schema:', error);
        }
    }

    function generateNodesFromSchema(schema: any): any[] {
        const nodes: any[] = [];
        let yPosition = 50;
        const xPosition = 50;
        const nodeSpacing = 100; // space between leaf nodes

        if (schema && schema.type === 'object' && schema.properties) {
            traverseSchema(schema, '', nodes, { x: xPosition, y: yPosition }, nodeSpacing);
        }

        return nodes;
    }

    function traverseSchema(
        schemaNode: any, 
        path: string, 
        nodes: any[], 
        position: { x: number, y: number }, 
        spacing: number
    ): { x: number, y: number } {
        if (!schemaNode || !schemaNode.properties) return position;

        Object.entries(schemaNode.properties).forEach(([key, value]: [string, any]) => {
            const currentPath = path ? `${path}.${key}` : key;
            
            if (value.type === 'object' && value.properties && !value.properties['#text']) {
                // complex component: add section node without handle
                const sectionNode = {
                    id: `schema-section-${currentPath}`,
                    type: 'sectionNode',
                    data: {
                        label: `Section: ${key}`,
                        description: `complex section: ${key}`,
                        type: 'section',
                        isSection: true
                    },
                    position: { x: position.x, y: position.y },
                    draggable: false,
                    selectable: false,
                    deletable: false,
                    style: 'width: 200px; height: 40px;'
                };
                nodes.push(sectionNode);
                position.y += 40; // space between sections

                // recursively traverse nested properties
                if (value.oneOf || value.anyOf || value.allOf) {
                    // handle choice components
                    const choiceSectionNode = {
                        id: `schema-choice-section-${currentPath}`,
                        type: 'sectionNode',
                        data: {
                            label: `Choice: ${key}`,
                            description: `choice section: ${key}`,
                            type: 'choice_section',
                            isSection: true
                        },
                        position: { x: position.x, y: position.y },
                        draggable: false,
                        selectable: false,
                        deletable: false,
                        style: 'width: 200px; height: 40px;'
                    };
                    nodes.push(choiceSectionNode);
                    position.y += 50; // space between sections
                } else {
                    position.y += 20; // space between section and first leaf
                    position = traverseSchema(value, currentPath, nodes, position, spacing);
                }

            } else if (value.type === 'object' && value.properties && value.properties['#text']) {
                // leaf node: with handle for connections
                const leafNode = {
                    id: `schema-leaf-${currentPath}`,
                    type: 'leafNode',
                    data: {
                        label: key,
                        description: value.properties['#text'].description || `metadata field: ${key}`,
                        type: value.properties['#text'].type || 'string',
                        format: value.properties['#text'].format || 'text',
                        required: false, // toggle to render required leaf nodes red
                        path: currentPath,
                        isLeaf: true,
                        // TODO FIX: for validation all metadata fields are outputs
                        is_input: false,
                        is_output: true,
                        target_variable: key
                    },
                    position: { x: position.x + 30, y: position.y }, // indentation
                    draggable: true,
                    selectable: true,
                    deletable: false,
                    style: 'width: 180px; height: 60px;'
                };
                nodes.push(leafNode);
                position.y += spacing;

            } else if (value.type === 'array' && value.items) {
                // array - section + leaf items
                const arraySectionNode = {
                    id: `schema-array-section-${currentPath}`,
                    type: 'sectionNode',
                    data: {
                        label: `Array: ${key}`,
                        type: 'array_section',
                        isSection: true
                    },
                    position: { x: position.x, y: position.y },
                    draggable: false,
                    selectable: false,
                    deletable: false,
                    style: 'width: 200px; height: 40px;'
                };
                nodes.push(arraySectionNode);
                position.y += 50;  // space between sections
                position.y += 20; // space between section and array items

                // process array items
                if (value.items.type === 'object' && value.items.properties) {
                    position = traverseSchema(value.items, currentPath, nodes, position, spacing);
                }
            }
        });

        return position;
    }

    loader(currentSchema);

    console.log('treecomponent: schema loaded, nodes generated');
</script>
