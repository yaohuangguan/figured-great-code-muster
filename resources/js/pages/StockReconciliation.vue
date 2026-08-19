<script setup>
import axios from 'axios';
import { computed, onMounted, ref } from 'vue';
import { shortDate } from '../format';

const classes = ref([]);
const records = ref([]);
const loading = ref(true);
const saving = ref(false);
const aiLoading = ref(false);
const aiProcessingId = ref(null); // specific record ID currently analyzing
const aiMessage = ref(null);
const aiError = ref(null);
const aiSummaryData = ref(null); // stores { summary, insights, importedMovements, skippedRecords, breakdown }

// Selected records for multi-select / bulk operations
const selectedRecordIds = ref([]);
const activeTab = ref('all'); // 'all', 'pending', 'imported'

// Map of recordId -> { status: 'pending'|'imported'|'skipped'|'analyzing', movementId: number|null, movement: object|null, reason: string|null }
const recordStatusMap = ref({});

const movementForm = ref({
    stock_class_id: null,
    type: 'sale',
    quantity: null,
    note: '',
    source_record_id: null,
});

onMounted(load);

async function load() {
    loading.value = true;
    try {
        const { data } = await axios.get('/api/stock');
        classes.value = data.classes;
        records.value = data.records;
        if (!movementForm.value.stock_class_id) {
            movementForm.value.stock_class_id = data.classes[0]?.id ?? null;
        }

        initializeRecordStatuses();
    } catch (e) {
        console.error('Failed to load stock data:', e);
    } finally {
        loading.value = false;
    }
}

// Match records with already keyed movements (e.g. the docking tally example)
function initializeRecordStatuses() {
    const map = {};
    for (const rec of records.value) {
        map[rec.id] = {
            status: 'pending',
            movementId: null,
            movement: null,
            reason: null,
        };
    }

    // Check all movements across classes to match them to records
    for (const sc of classes.value) {
        for (const mv of sc.movements) {
            // Check if note references a record ID or docket/diary date
            const idMatch = mv.note?.match(/\[REC-(\d+)\]/);
            if (idMatch && map[idMatch[1]]) {
                map[idMatch[1]] = {
                    status: 'imported',
                    movementId: mv.id,
                    movement: { ...mv, stock_class_name: sc.name },
                    reason: null,
                };
                continue;
            }

            // Match the default seeded docking tally
            if (mv.type === 'birth' && mv.quantity === 1240 && mv.note?.toLowerCase().includes('docking')) {
                const dockingRecord = records.value.find((r) => r.body.toLowerCase().includes('docked 1,240 lambs'));
                if (dockingRecord && map[dockingRecord.id]) {
                    map[dockingRecord.id] = {
                        status: 'imported',
                        movementId: mv.id,
                        movement: { ...mv, stock_class_name: sc.name },
                        reason: 'Docking tally (Opening Example)',
                    };
                }
            }
        }
    }

    recordStatusMap.value = map;
}

// Tally per class: opening + births + purchases - deaths - sales.
function tally(stockClass) {
    const sum = (type) =>
        stockClass.movements.filter((m) => m.type === type).reduce((total, m) => total + m.quantity, 0);
    const calculated = stockClass.opening_count + sum('birth') + sum('purchase') - sum('death') - sum('sale');
    return {
        births: sum('birth'),
        purchases: sum('purchase'),
        deaths: sum('death'),
        sales: sum('sale'),
        calculated,
        difference: calculated - stockClass.closing_count,
    };
}

const canSave = computed(
    () => movementForm.value.stock_class_id && movementForm.value.quantity > 0 && movementForm.value.type,
);

async function addMovement() {
    saving.value = true;
    try {
        const sourceRecId = movementForm.value.source_record_id;
        const noteWithRec = sourceRecId
            ? `${movementForm.value.note ? movementForm.value.note + ' ' : ''}[REC-${sourceRecId}]`
            : movementForm.value.note;

        const { data } = await axios.post('/api/stock-movements', {
            stock_class_id: movementForm.value.stock_class_id,
            type: movementForm.value.type,
            quantity: movementForm.value.quantity,
            note: noteWithRec,
        });

        const targetClass = classes.value.find((c) => c.id === data.stock_class_id);
        if (targetClass) {
            targetClass.movements.push(data);
        }

        if (sourceRecId && recordStatusMap.value[sourceRecId]) {
            recordStatusMap.value[sourceRecId] = {
                status: 'imported',
                movementId: data.id,
                movement: { ...data, stock_class_name: targetClass?.name },
                reason: null,
            };
        }

        // Reset form
        movementForm.value.quantity = null;
        movementForm.value.note = '';
        movementForm.value.source_record_id = null;
    } catch (e) {
        aiError.value = e.response?.data?.message ?? e.message;
    } finally {
        saving.value = false;
    }
}

async function removeMovement(stockClass, movement) {
    await axios.delete(`/api/stock-movements/${movement.id}`);
    stockClass.movements = stockClass.movements.filter((m) => m.id !== movement.id);

    // Revert any matching record back to pending
    for (const [recId, entry] of Object.entries(recordStatusMap.value)) {
        if (entry.movementId === movement.id) {
            recordStatusMap.value[recId] = {
                status: 'pending',
                movementId: null,
                movement: null,
                reason: null,
            };
        }
    }
}

function prefillRecordToForm(record) {
    movementForm.value.source_record_id = record.id;
    movementForm.value.note = `${record.source} ${shortDate(record.recorded_on)}`;

    // Simple heuristic prefill
    const body = record.body.toLowerCase();
    if (body.includes('ewe') || body.includes('two-tooth')) {
        const ewes = classes.value.find((c) => c.name.toLowerCase().includes('ewe'));
        if (ewes) movementForm.value.stock_class_id = ewes.id;
    } else if (body.includes('lamb')) {
        const lambs = classes.value.find((c) => c.name.toLowerCase().includes('lamb'));
        if (lambs) movementForm.value.stock_class_id = lambs.id;
    } else if (body.includes('cow') || body.includes('calv') || body.includes('steer') || body.includes('cattle')) {
        const cattle = classes.value.find((c) => c.name.toLowerCase().includes('cattle'));
        if (cattle) movementForm.value.stock_class_id = cattle.id;
    }

    if (body.includes('dock') || body.includes('calv') || body.includes('born') || body.includes('on the ground')) {
        movementForm.value.type = 'birth';
    } else if (body.includes('purchase') || body.includes('bought')) {
        movementForm.value.type = 'purchase';
    } else if (body.includes('dead') || body.includes('died') || body.includes('lost') || body.includes('down') || body.includes('killed')) {
        movementForm.value.type = 'death';
    } else if (body.includes('sale') || body.includes('sold')) {
        movementForm.value.type = 'sale';
    }

    // Scroll to top left
    window.scrollTo({ top: 0, behavior: 'smooth' });
}

// Filtered paper trail records
const filteredRecords = computed(() => {
    if (activeTab.value === 'pending') {
        return records.value.filter((r) => recordStatusMap.value[r.id]?.status === 'pending');
    }
    if (activeTab.value === 'imported') {
        return records.value.filter((r) => recordStatusMap.value[r.id]?.status === 'imported');
    }
    return records.value;
});

// Selection computations
const allFilteredSelected = computed(() => {
    if (!filteredRecords.value.length) return false;
    return filteredRecords.value.every((r) => selectedRecordIds.value.includes(r.id));
});

const isIndeterminate = computed(() => {
    const count = filteredRecords.value.filter((r) => selectedRecordIds.value.includes(r.id)).length;
    return count > 0 && count < filteredRecords.value.length;
});

function toggleSelectAll() {
    if (allFilteredSelected.value) {
        // Deselect current filtered
        const filteredIds = new Set(filteredRecords.value.map((r) => r.id));
        selectedRecordIds.value = selectedRecordIds.value.filter((id) => !filteredIds.has(id));
    } else {
        // Add all filtered
        const union = new Set([...selectedRecordIds.value, ...filteredRecords.value.map((r) => r.id)]);
        selectedRecordIds.value = Array.from(union);
    }
}

function toggleRecordSelect(id) {
    if (selectedRecordIds.value.includes(id)) {
        selectedRecordIds.value = selectedRecordIds.value.filter((item) => item !== id);
    } else {
        selectedRecordIds.value.push(id);
    }
}

// Counts
const pendingCount = computed(
    () => records.value.filter((r) => recordStatusMap.value[r.id]?.status === 'pending').length,
);
const importedCount = computed(
    () => records.value.filter((r) => recordStatusMap.value[r.id]?.status === 'imported').length,
);

// AI Prompting and Reconciliation
async function runAiReconciliation(recordsToProcess) {
    if (!recordsToProcess || recordsToProcess.length === 0) return;

    aiLoading.value = true;
    aiError.value = null;
    aiMessage.value = null;

    // Mark processing state
    recordsToProcess.forEach((r) => {
        if (recordStatusMap.value[r.id]) {
            recordStatusMap.value[r.id].status = 'analyzing';
        }
    });

    const classInfo = classes.value.map((c) => ({ id: c.id, name: c.name }));

    const systemPrompt = `You are a specialist agricultural accounting assistant for New Zealand livestock farm reconciliations.
Your task is to analyze farm records (diaries, sale/purchase dockets, text messages) for Kahikatea Downs (Stock year 1 Jul 2025 – 30 Jun 2026) and extract exact stock movement events.

Available Stock Classes:
${JSON.stringify(classInfo, null, 2)}

Movement Types:
- "birth": Lambing, calving, docking tallies, calves on the ground.
- "purchase": Bought stock, purchase dockets.
- "death": Stock that died, lost in storms/bog/drought, bearing trouble, euthanized, or killed for farm meat freezer.
- "sale": Sold stock, sale dockets, livestock trucks away, private sales.

Crucial Business Logic & Farm Nuances:
1. "Ewes", "two-tooth ewes", "cull ewes" map to "Ewes".
2. "Lambs" map to "Lambs".
3. "Cows", "calves", "steers", "R2 steers", "cull cows" map to "Cattle".
4. Text message updates/corrections: If an earlier message is corrected by a later message (e.g. Kate correcting "sold 40 cull cows" to "neighbour only ended up taking 38, we kept 2 back"), only the final correct quantity (38 sale) should be imported. Mark the earlier superseded message with is_movement: false.
5. Repeated / duplicate dockets: If identical dockets appear (e.g. S-40417 on both Feb 17 and Feb 24), the duplicate should have is_movement: false with reasoning explaining it is a duplicate docket.
6. Confirmations: A diary entry like "Steers away on the truck this morning - 38 head" that confirms a sale docket (S-40488) on the same batch should not be double-counted if the docket is already counted (or count the docket and mark the diary confirmation as is_movement: false).
7. "Docked 1,240 lambs" (6 Oct 2025) is already seeded as the opening worked example. Mark is_already_imported: true or is_movement: false.

Output ONLY valid JSON in this exact structure:
{
  "summary": "Brief 1-2 sentence overview of the reconciliation actions performed",
  "insights": [
    "Key farm insight 1 (e.g. drought impact, weather losses, sale averages, etc.)",
    "Key farm insight 2 (e.g. docket deduplication or correction notes applied)"
  ],
  "movements": [
    {
      "record_id": 1,
      "is_movement": true,
      "stock_class_id": 1,
      "type": "death",
      "quantity": 2,
      "note": "Bearing trouble (Diary 14 Aug 2025)",
      "reasoning": "2 ewes euthanized with bearing trouble"
    }
  ]
}`;

    const userPrompt = `Please analyze the following farm records, extract stock movements, and generate a summary with insights:
${JSON.stringify(recordsToProcess.map((r) => ({ id: r.id, date: r.recorded_on, source: r.source, text: r.body })), null, 2)}`;

    try {
        const { data } = await axios.post('/api/ai', {
            system: systemPrompt,
            prompt: userPrompt,
        });

        let jsonText = data.text.trim();
        // Strip markdown code block wrappers if any
        if (jsonText.startsWith('```')) {
            jsonText = jsonText.replace(/^```(json)?\n?/, '').replace(/\n?```$/, '');
        }

        const parsedResponse = JSON.parse(jsonText);
        const movementsList = Array.isArray(parsedResponse) ? parsedResponse : (parsedResponse.movements || []);
        const summaryText = parsedResponse.summary || `Successfully processed ${recordsToProcess.length} paper trail record(s).`;
        const insightsList = parsedResponse.insights || [];

        let newlyImportedList = [];
        let newlySkippedList = [];
        const classBreakdown = {};

        for (const item of movementsList) {
            const rec = records.value.find((r) => r.id === item.record_id);
            if (!rec) continue;

            if (item.is_movement && item.stock_class_id && item.quantity > 0) {
                // Check if this record is already imported
                if (recordStatusMap.value[rec.id]?.status === 'imported') {
                    continue;
                }

                const noteWithTag = `${item.note || rec.source} [REC-${rec.id}]`;
                const { data: createdMovement } = await axios.post('/api/stock-movements', {
                    stock_class_id: item.stock_class_id,
                    type: item.type,
                    quantity: item.quantity,
                    note: noteWithTag,
                });

                const sc = classes.value.find((c) => c.id === item.stock_class_id);
                if (sc) {
                    sc.movements.push(createdMovement);
                    if (!classBreakdown[sc.name]) {
                        classBreakdown[sc.name] = { births: 0, purchases: 0, deaths: 0, sales: 0 };
                    }
                    if (classBreakdown[sc.name][item.type + 's'] !== undefined) {
                        classBreakdown[sc.name][item.type + 's'] += item.quantity;
                    }
                }

                recordStatusMap.value[rec.id] = {
                    status: 'imported',
                    movementId: createdMovement.id,
                    movement: { ...createdMovement, stock_class_name: sc?.name },
                    reason: item.reasoning,
                };
                newlyImportedList.push({
                    record: rec,
                    movement: createdMovement,
                    stock_class_name: sc?.name,
                    reasoning: item.reasoning,
                });
            } else {
                recordStatusMap.value[rec.id] = {
                    status: 'skipped',
                    movementId: null,
                    movement: null,
                    reason: item.reasoning || 'No movement / Duplicate / Superseded',
                };
                newlySkippedList.push({
                    record: rec,
                    reasoning: item.reasoning || 'Duplicate or non-movement log',
                });
            }
        }

        // Set rich summary data
        aiSummaryData.value = {
            timestamp: new Date().toLocaleTimeString(),
            summary: summaryText,
            insights: insightsList,
            importedCount: newlyImportedList.length,
            skippedCount: newlySkippedList.length,
            importedMovements: newlyImportedList,
            skippedRecords: newlySkippedList,
            classBreakdown,
        };

        aiMessage.value = `AI Analysis Complete: Imported ${newlyImportedList.length} movement(s)${newlySkippedList.length ? ` and handled ${newlySkippedList.length} duplicate/skipped record(s)` : ''}.`;
        // Clear selection after batch run
        selectedRecordIds.value = [];
    } catch (e) {
        console.error('AI reconciliation failed:', e);
        aiError.value = e.response?.data?.error || e.message || 'AI analysis request failed. Please check network/key.';
        // Revert analyzing statuses back to pending
        recordsToProcess.forEach((r) => {
            if (recordStatusMap.value[r.id]?.status === 'analyzing') {
                recordStatusMap.value[r.id].status = 'pending';
            }
        });
    } finally {
        aiLoading.value = false;
        aiProcessingId.value = null;
    }
}

async function analyzeSingleRecord(record) {
    aiProcessingId.value = record.id;
    await runAiReconciliation([record]);
}

async function autoReconcileSelected() {
    const selectedRecs = records.value.filter((r) => selectedRecordIds.value.includes(r.id));
    if (selectedRecs.length === 0) return;
    await runAiReconciliation(selectedRecs);
}

async function autoReconcileAllPending() {
    const pendingRecs = records.value.filter((r) => recordStatusMap.value[r.id]?.status === 'pending');
    if (pendingRecs.length === 0) return;
    await runAiReconciliation(pendingRecs);
}

const sourceBadgeClass = {
    'Diary': 'bg-fg-warning-15 text-fg-warning-text border border-fg-warning-15',
    'Sale docket': 'bg-fg-light-blue-15 text-fg-light-blue border border-fg-light-blue-15',
    'Text message': 'bg-fg-brown-15 text-fg-brown border border-fg-brown-15',
};

const movementTypeBadgeClass = {
    'birth': 'bg-fg-positive-15 text-fg-positive-dark',
    'purchase': 'bg-fg-main-blue-15 text-fg-main-blue',
    'death': 'bg-fg-danger-15 text-fg-danger-dark',
    'sale': 'bg-fg-warning-15 text-fg-warning-text',
};
</script>

<template>
    <div class="space-y-4">
        <!-- Page Title & Header -->
        <div class="flex flex-col gap-2 md:flex-row md:items-center md:justify-between">
            <div>
                <h2 class="text-xl font-semibold tracking-tight text-fg-dark-blue">Stock reconciliation — Kahikatea Downs</h2>
                <p class="text-sm text-fg-mid-grey">
                    Key stock movements in from raw records (right) until each tally matches the farmer's recorded closing count.
                    Stock year 1 Jul 2025 – 30 Jun 2026.
                </p>
            </div>
            <!-- Overall reconciliation status pill -->
            <div
                v-if="!loading"
                class="flex items-center gap-2 rounded-lg border px-3 py-1.5 text-xs font-medium shadow-xs"
                :class="
                    classes.every((c) => tally(c).difference === 0)
                        ? 'border-fg-positive bg-fg-positive-9 text-fg-positive-dark'
                        : 'border-fg-warning bg-fg-warning-15 text-fg-warning-text'
                "
            >
                <span
                    class="h-2 w-2 rounded-full"
                    :class="classes.every((c) => tally(c).difference === 0) ? 'bg-fg-positive' : 'bg-fg-warning animate-ping'"
                ></span>
                <span>
                    {{
                        classes.every((c) => tally(c).difference === 0)
                            ? 'All stock classes fully reconciled ✓'
                            : `${classes.filter((c) => tally(c).difference !== 0).length} class(es) pending reconciliation`
                    }}
                </span>
            </div>
        </div>

        <p v-if="loading" class="text-sm text-fg-light-grey">Loading stock data…</p>

        <div v-else class="grid grid-cols-1 gap-5 lg:grid-cols-12">
            <!-- LEFT COLUMN (5 cols on lg): Key in a movement (TOP) + Tally tallies (BELOW) -->
            <div class="space-y-4 lg:col-span-5">
                <!-- 1. Key in a movement form (MOVED TO TOP BELOW TITLE) -->
                <div class="rounded-lg border border-fg-muted-grey bg-white p-4 shadow-xs">
                    <div class="mb-3 flex items-center justify-between">
                        <div class="flex items-center gap-2">
                            <div class="flex h-6 w-6 items-center justify-center rounded bg-fg-main-blue-15 text-xs font-bold text-fg-main-blue">
                                ✎
                            </div>
                            <h3 class="text-sm font-semibold text-fg-dark-blue">Key in a movement</h3>
                        </div>
                        <span v-if="movementForm.source_record_id" class="rounded bg-fg-main-blue-9 px-2 py-0.5 text-xs font-medium text-fg-main-blue">
                            Prefilled from #{{ movementForm.source_record_id }}
                        </span>
                    </div>

                    <div class="space-y-3">
                        <div class="grid grid-cols-2 gap-2">
                            <div>
                                <label class="block text-xs font-medium text-fg-mid-grey">Stock class</label>
                                <select
                                    v-model="movementForm.stock_class_id"
                                    class="mt-1 w-full rounded border border-fg-muted-grey bg-white px-2 py-1.5 text-sm transition focus:border-fg-main-blue"
                                >
                                    <option v-for="c in classes" :key="c.id" :value="c.id">{{ c.name }}</option>
                                </select>
                            </div>
                            <div>
                                <label class="block text-xs font-medium text-fg-mid-grey">Type</label>
                                <select
                                    v-model="movementForm.type"
                                    class="mt-1 w-full rounded border border-fg-muted-grey bg-white px-2 py-1.5 text-sm capitalize transition focus:border-fg-main-blue"
                                >
                                    <option value="birth">Birth (+)</option>
                                    <option value="purchase">Purchase (+)</option>
                                    <option value="death">Death (−)</option>
                                    <option value="sale">Sale (−)</option>
                                </select>
                            </div>
                        </div>

                        <div class="grid grid-cols-3 gap-2">
                            <div>
                                <label class="block text-xs font-medium text-fg-mid-grey">Quantity</label>
                                <input
                                    v-model.number="movementForm.quantity"
                                    type="number"
                                    min="1"
                                    placeholder="e.g. 20"
                                    class="mt-1 w-full rounded border border-fg-muted-grey px-2 py-1.5 text-right font-mono text-sm transition focus:border-fg-main-blue"
                                />
                            </div>
                            <div class="col-span-2">
                                <label class="block text-xs font-medium text-fg-mid-grey">Note / Source reference</label>
                                <input
                                    v-model="movementForm.note"
                                    placeholder="e.g. docket S-40102"
                                    class="mt-1 w-full rounded border border-fg-muted-grey px-2 py-1.5 text-sm transition focus:border-fg-main-blue"
                                />
                            </div>
                        </div>

                        <div class="flex items-center justify-between pt-1">
                            <button
                                v-if="movementForm.source_record_id"
                                type="button"
                                class="text-xs text-fg-mid-grey hover:text-fg-danger"
                                @click="movementForm.source_record_id = null; movementForm.note = ''; movementForm.quantity = null"
                            >
                                Clear prefill
                            </button>
                            <span v-else></span>

                            <button
                                class="inline-flex items-center gap-1.5 rounded bg-fg-main-blue px-4 py-1.5 text-sm font-medium text-white shadow-xs hover:bg-fg-main-blue-hover disabled:opacity-50"
                                :disabled="!canSave || saving"
                                @click="addMovement"
                            >
                                <span>{{ saving ? 'Saving…' : 'Add Movement' }}</span>
                            </button>
                        </div>
                    </div>
                </div>

                <!-- 2. Stock Class Tallies Cards -->
                <div
                    v-for="stockClass in classes"
                    :key="stockClass.id"
                    class="rounded-lg border border-fg-muted-grey bg-white p-4 shadow-xs transition hover:border-fg-main-blue-30"
                >
                    <div class="mb-2 flex items-center justify-between">
                        <div class="flex items-center gap-2">
                            <h3 class="font-semibold text-fg-dark-blue">{{ stockClass.name }}</h3>
                            <span class="text-xs text-fg-light-grey">
                                ({{ stockClass.movements.length }} movement{{ stockClass.movements.length === 1 ? '' : 's' }})
                            </span>
                        </div>
                        <span
                            class="rounded-full px-2.5 py-0.5 text-xs font-semibold uppercase tracking-wide"
                            :class="
                                tally(stockClass).difference === 0
                                    ? 'bg-fg-positive-15 text-fg-positive-dark'
                                    : 'bg-fg-danger-15 text-fg-danger-dark'
                            "
                        >
                            {{
                                tally(stockClass).difference === 0
                                    ? '✓ Reconciled'
                                    : `Out by ${tally(stockClass).difference > 0 ? '+' : ''}${tally(stockClass).difference}`
                            }}
                        </span>
                    </div>

                    <table class="w-full text-sm">
                        <tbody>
                            <tr class="border-t border-fg-pale-grey">
                                <td class="py-1 text-fg-mid-grey">Opening (1 Jul 2025)</td>
                                <td class="py-1 text-right font-mono">{{ stockClass.opening_count.toLocaleString() }}</td>
                            </tr>
                            <tr class="border-t border-fg-pale-grey">
                                <td class="py-1 text-fg-mid-grey">+ Births</td>
                                <td class="py-1 text-right font-mono text-fg-positive-dark">
                                    {{ tally(stockClass).births ? `+${tally(stockClass).births.toLocaleString()}` : '0' }}
                                </td>
                            </tr>
                            <tr class="border-t border-fg-pale-grey">
                                <td class="py-1 text-fg-mid-grey">+ Purchases</td>
                                <td class="py-1 text-right font-mono text-fg-main-blue">
                                    {{ tally(stockClass).purchases ? `+${tally(stockClass).purchases.toLocaleString()}` : '0' }}
                                </td>
                            </tr>
                            <tr class="border-t border-fg-pale-grey">
                                <td class="py-1 text-fg-mid-grey">− Deaths</td>
                                <td class="py-1 text-right font-mono text-fg-danger-dark">
                                    {{ tally(stockClass).deaths ? `−${tally(stockClass).deaths.toLocaleString()}` : '0' }}
                                </td>
                            </tr>
                            <tr class="border-t border-fg-pale-grey">
                                <td class="py-1 text-fg-mid-grey">− Sales</td>
                                <td class="py-1 text-right font-mono text-fg-warning-text">
                                    {{ tally(stockClass).sales ? `−${tally(stockClass).sales.toLocaleString()}` : '0' }}
                                </td>
                            </tr>
                            <tr class="border-t-2 border-fg-muted-grey font-semibold text-fg-dark-blue">
                                <td class="py-1">= Calculated closing</td>
                                <td class="py-1 text-right font-mono">{{ tally(stockClass).calculated.toLocaleString() }}</td>
                            </tr>
                            <tr class="border-t border-fg-pale-grey text-xs">
                                <td class="py-1 text-fg-mid-grey">Recorded closing (tally book)</td>
                                <td class="py-1 text-right font-mono font-medium">{{ stockClass.closing_count.toLocaleString() }}</td>
                            </tr>
                        </tbody>
                    </table>

                    <!-- Movement list dropdown per class -->
                    <details v-if="stockClass.movements.length" class="mt-3 rounded border border-fg-pale-grey bg-fg-super-pale-grey p-2" open>
                        <summary class="cursor-pointer text-xs font-medium text-fg-mid-grey hover:text-fg-dark-blue">
                            Entered movements ({{ stockClass.movements.length }})
                        </summary>
                        <ul class="mt-2 space-y-1.5">
                            <li
                                v-for="movement in stockClass.movements"
                                :key="movement.id"
                                class="flex items-center justify-between rounded border border-fg-muted-grey/60 bg-white px-2.5 py-1 text-xs shadow-2xs"
                            >
                                <div class="flex items-center gap-1.5">
                                    <span
                                        class="rounded px-1.5 py-0.5 text-[10px] font-semibold uppercase"
                                        :class="movementTypeBadgeClass[movement.type]"
                                    >
                                        {{ movement.type }}
                                    </span>
                                    <span class="font-mono font-semibold text-fg-dark-blue">
                                        × {{ movement.quantity.toLocaleString() }}
                                    </span>
                                    <span v-if="movement.note" class="truncate max-w-[170px] text-fg-mid-grey" :title="movement.note">
                                        — {{ movement.note }}
                                    </span>
                                </div>
                                <button
                                    class="rounded p-0.5 text-fg-light-grey hover:bg-fg-danger-15 hover:text-fg-danger"
                                    title="Delete movement"
                                    @click="removeMovement(stockClass, movement)"
                                >
                                    ✕
                                </button>
                            </li>
                        </ul>
                    </details>
                </div>
            </div>

            <!-- RIGHT COLUMN (7 cols on lg): The paper trail with multi-select and AI analysis -->
            <div class="space-y-3 lg:col-span-7">
                <!-- AI SUMMARY & INSIGHTS CARD (ABOVE PAPER TRAIL, MANUALLY DISMISSIBLE) -->
                <div
                    v-if="aiSummaryData"
                    class="overflow-hidden rounded-lg border border-fg-main-blue-30 bg-white shadow-sm transition-all"
                >
                    <!-- Summary Card Header -->
                    <div class="flex items-center justify-between border-b border-fg-main-blue-15 bg-fg-main-blue-9 px-4 py-3">
                        <div class="flex items-center gap-2">
                            <span class="flex h-6 w-6 items-center justify-center rounded-full bg-fg-main-blue text-xs text-white">✨</span>
                            <div>
                                <h3 class="text-sm font-bold text-fg-dark-blue">AI Reconciliation Summary & Insights</h3>
                                <span class="text-[11px] text-fg-mid-grey">Generated at {{ aiSummaryData.timestamp }}</span>
                            </div>
                        </div>
                        <div class="flex items-center gap-2">
                            <span class="rounded-full bg-fg-positive-15 px-2 py-0.5 text-xs font-semibold text-fg-positive-dark">
                                +{{ aiSummaryData.importedCount }} movements imported
                            </span>
                            <button
                                class="rounded p-1 text-fg-mid-grey hover:bg-fg-main-blue-15 hover:text-fg-dark-blue transition cursor-pointer"
                                title="Close summary"
                                @click="aiSummaryData = null"
                            >
                                ✕
                            </button>
                        </div>
                    </div>

                    <!-- Summary Content Body -->
                    <div class="p-4 space-y-3.5 text-xs">
                        <!-- Overview narrative -->
                        <div class="rounded-md bg-fg-super-pale-grey p-3 border border-fg-pale-grey">
                            <p class="font-medium text-fg-dark-blue leading-relaxed">
                                {{ aiSummaryData.summary }}
                            </p>
                        </div>

                        <!-- What was imported (Breakdown) -->
                        <div>
                            <h4 class="font-semibold text-fg-dark-blue mb-1.5 flex items-center gap-1.5">
                                <span>📊</span> What AI Imported & Categorized:
                            </h4>
                            <div class="grid grid-cols-1 sm:grid-cols-3 gap-2">
                                <div
                                    v-for="(breakdown, className) in aiSummaryData.classBreakdown"
                                    :key="className"
                                    class="rounded border border-fg-muted-grey/60 bg-white p-2.5 shadow-2xs"
                                >
                                    <div class="font-bold text-fg-dark-blue border-b border-fg-pale-grey pb-1 mb-1.5 flex justify-between">
                                        <span>{{ className }}</span>
                                    </div>
                                    <div class="space-y-0.5 text-[11px]">
                                        <div v-if="breakdown.births" class="text-fg-positive-dark">
                                            Births: +{{ breakdown.births.toLocaleString() }}
                                        </div>
                                        <div v-if="breakdown.purchases" class="text-fg-main-blue">
                                            Purchases: +{{ breakdown.purchases.toLocaleString() }}
                                        </div>
                                        <div v-if="breakdown.deaths" class="text-fg-danger-dark">
                                            Deaths: −{{ breakdown.deaths.toLocaleString() }}
                                        </div>
                                        <div v-if="breakdown.sales" class="text-fg-warning-text">
                                            Sales: −{{ breakdown.sales.toLocaleString() }}
                                        </div>
                                    </div>
                                </div>
                            </div>
                        </div>

                        <!-- Key Insights / Observations -->
                        <div v-if="aiSummaryData.insights && aiSummaryData.insights.length > 0">
                            <h4 class="font-semibold text-fg-dark-blue mb-1.5 flex items-center gap-1.5">
                                <span>💡</span> Key Farm Insights & Deduplications:
                            </h4>
                            <ul class="space-y-1 rounded border border-fg-warning-15 bg-fg-warning-15/40 p-2.5 text-[11px] text-fg-dark-grey">
                                <li
                                    v-for="(insight, idx) in aiSummaryData.insights"
                                    :key="idx"
                                    class="flex items-start gap-1.5 leading-snug"
                                >
                                    <span class="text-fg-warning-text font-bold mt-0.5">•</span>
                                    <span>{{ insight }}</span>
                                </li>
                            </ul>
                        </div>

                        <!-- Footer dismiss button -->
                        <div class="flex justify-end pt-1">
                            <button
                                class="rounded bg-fg-pale-grey px-3 py-1 text-xs font-medium text-fg-dark-grey hover:bg-fg-muted-grey transition cursor-pointer"
                                @click="aiSummaryData = null"
                            >
                                Dismiss Summary
                            </button>
                        </div>
                    </div>
                </div>

                <!-- AI Status Alert Banner -->
                <div v-if="aiMessage && !aiSummaryData" class="flex items-center justify-between rounded-lg bg-fg-positive-9 border border-fg-positive-15 p-3 text-xs text-fg-positive-dark">
                    <div class="flex items-center gap-2">
                        <span>✨</span>
                        <span>{{ aiMessage }}</span>
                    </div>
                    <button class="text-fg-positive-dark font-bold hover:underline" @click="aiMessage = null">✕</button>
                </div>

                <div v-if="aiError" class="flex items-center justify-between rounded-lg bg-fg-danger-9 border border-fg-danger-15 p-3 text-xs text-fg-danger-dark">
                    <div class="flex items-center gap-2">
                        <span>⚠️</span>
                        <span>{{ aiError }}</span>
                    </div>
                    <button class="text-fg-danger-dark font-bold hover:underline" @click="aiError = null">✕</button>
                </div>

                <!-- Paper Trail Main Container -->
                <div class="overflow-hidden rounded-lg border border-fg-muted-grey bg-white shadow-xs">
                    <!-- Header & Bulk Action Bar -->
                    <div class="border-b border-fg-pale-grey bg-fg-super-pale-grey p-3 sm:p-4">
                        <div class="flex flex-col gap-2 sm:flex-row sm:items-center sm:justify-between">
                            <div>
                                <div class="flex items-center gap-2">
                                    <h3 class="text-sm font-bold text-fg-dark-blue">The paper trail</h3>
                                    <span class="rounded-full bg-fg-main-blue-15 px-2 py-0.5 text-[11px] font-medium text-fg-main-blue">
                                        {{ records.length }} logs
                                    </span>
                                </div>
                                <p class="text-xs text-fg-mid-grey">
                                    {{ importedCount }} imported • {{ pendingCount }} pending reconciliation
                                </p>
                            </div>

                            <!-- AI Bulk Actions -->
                            <div class="flex flex-wrap items-center gap-2">
                                <button
                                    v-if="selectedRecordIds.length > 0"
                                    class="inline-flex items-center gap-1.5 rounded bg-fg-main-blue px-3 py-1.5 text-xs font-semibold text-white shadow-xs hover:bg-fg-main-blue-hover disabled:opacity-50 cursor-pointer"
                                    :disabled="aiLoading"
                                    @click="autoReconcileSelected"
                                >
                                    <span>✨</span>
                                    <span>AI Reconcile Selected ({{ selectedRecordIds.length }})</span>
                                </button>

                                <button
                                    v-else
                                    class="inline-flex items-center gap-1.5 rounded border border-fg-main-blue bg-fg-main-blue-9 px-3 py-1.5 text-xs font-semibold text-fg-main-blue hover:bg-fg-main-blue-15 disabled:opacity-50 cursor-pointer"
                                    :disabled="aiLoading || pendingCount === 0"
                                    @click="autoReconcileAllPending"
                                >
                                    <span>✨</span>
                                    <span>{{ aiLoading ? 'AI Reconciling…' : `AI Auto-Reconcile All (${pendingCount})` }}</span>
                                </button>
                            </div>
                        </div>

                        <!-- Filter Tabs & Bulk Select Bar -->
                        <div class="mt-3 flex items-center justify-between border-t border-fg-muted-grey/50 pt-2.5 text-xs">
                            <div class="flex items-center gap-1">
                                <button
                                    class="rounded px-2.5 py-1 font-medium transition cursor-pointer"
                                    :class="activeTab === 'all' ? 'bg-white font-semibold text-fg-dark-blue shadow-xs' : 'text-fg-mid-grey hover:text-fg-dark-blue'"
                                    @click="activeTab = 'all'"
                                >
                                    All ({{ records.length }})
                                </button>
                                <button
                                    class="rounded px-2.5 py-1 font-medium transition cursor-pointer"
                                    :class="activeTab === 'pending' ? 'bg-white font-semibold text-fg-warning-text shadow-xs' : 'text-fg-mid-grey hover:text-fg-dark-blue'"
                                    @click="activeTab = 'pending'"
                                >
                                    Pending ({{ pendingCount }})
                                </button>
                                <button
                                    class="rounded px-2.5 py-1 font-medium transition cursor-pointer"
                                    :class="activeTab === 'imported' ? 'bg-white font-semibold text-fg-positive-dark shadow-xs' : 'text-fg-mid-grey hover:text-fg-dark-blue'"
                                    @click="activeTab = 'imported'"
                                >
                                    Imported ({{ importedCount }})
                                </button>
                            </div>

                            <!-- Bulk Select Checkbox Controls -->
                            <div class="flex items-center gap-2">
                                <label class="inline-flex cursor-pointer items-center gap-1.5 font-medium text-fg-dark-grey hover:text-fg-dark-blue">
                                    <input
                                        type="checkbox"
                                        :checked="allFilteredSelected"
                                        :indeterminate="isIndeterminate"
                                        class="h-4 w-4 rounded border-fg-muted-grey text-fg-main-blue focus:ring-fg-main-blue cursor-pointer"
                                        @change="toggleSelectAll"
                                    />
                                    <span>Bulk Select</span>
                                </label>
                                <span v-if="selectedRecordIds.length" class="text-fg-light-grey">
                                    ({{ selectedRecordIds.length }} selected)
                                </span>
                            </div>
                        </div>
                    </div>

                    <!-- Records Table / Listing -->
                    <div class="overflow-x-auto">
                        <table class="w-full text-left text-xs">
                            <thead class="border-b border-fg-pale-grey bg-fg-super-pale-grey text-fg-mid-grey">
                                <tr>
                                    <th class="w-10 px-3 py-2 text-center">
                                        <input
                                            type="checkbox"
                                            :checked="allFilteredSelected"
                                            :indeterminate="isIndeterminate"
                                            class="h-3.5 w-3.5 rounded border-fg-muted-grey text-fg-main-blue focus:ring-fg-main-blue cursor-pointer"
                                            @change="toggleSelectAll"
                                        />
                                    </th>
                                    <th class="w-14 px-2 py-2 font-semibold">Key</th>
                                    <th class="w-24 px-2 py-2 font-semibold">Date</th>
                                    <th class="w-28 px-2 py-2 font-semibold">Source</th>
                                    <th class="px-3 py-2 font-semibold">Paper Trail Log</th>
                                    <th class="w-28 px-2 py-2 font-semibold">Status</th>
                                    <th class="w-28 px-3 py-2 text-right font-semibold">Action</th>
                                </tr>
                            </thead>
                            <tbody class="divide-y divide-fg-pale-grey">
                                <tr
                                    v-for="record in filteredRecords"
                                    :key="record.id"
                                    class="transition hover:bg-fg-super-pale-grey/80"
                                    :class="[
                                        selectedRecordIds.includes(record.id) ? 'bg-fg-main-blue-9/40' : '',
                                        recordStatusMap[record.id]?.status === 'imported' ? 'bg-fg-positive-9/20' : '',
                                    ]"
                                >
                                    <!-- Selection checkbox -->
                                    <td class="px-3 py-2.5 text-center">
                                        <input
                                            type="checkbox"
                                            :checked="selectedRecordIds.includes(record.id)"
                                            class="h-3.5 w-3.5 rounded border-fg-muted-grey text-fg-main-blue focus:ring-fg-main-blue cursor-pointer"
                                            @change="toggleRecordSelect(record.id)"
                                        />
                                    </td>

                                    <!-- Unique Key / ID -->
                                    <td class="px-2 py-2.5 font-mono text-[11px] font-bold text-fg-mid-grey">
                                        #{{ record.id }}
                                    </td>

                                    <!-- Date -->
                                    <td class="whitespace-nowrap px-2 py-2.5 text-fg-mid-grey">
                                        {{ shortDate(record.recorded_on) }}
                                    </td>

                                    <!-- Source Badge -->
                                    <td class="px-2 py-2.5">
                                        <span class="rounded-full px-2 py-0.5 text-[10px] font-medium" :class="sourceBadgeClass[record.source]">
                                            {{ record.source }}
                                        </span>
                                    </td>

                                    <!-- Log content & details -->
                                    <td class="px-3 py-2.5">
                                        <p class="leading-snug text-fg-dark-grey">{{ record.body }}</p>
                                        <!-- If movement details or reason exist -->
                                        <div
                                            v-if="recordStatusMap[record.id]?.movement"
                                            class="mt-1 flex items-center gap-1.5 text-[11px] text-fg-positive-dark font-medium"
                                        >
                                            <span>✓ Imported as:</span>
                                            <span class="rounded bg-fg-positive-15 px-1.5 py-0.5 font-mono text-[10px]">
                                                {{ recordStatusMap[record.id].movement.stock_class_name }} ·
                                                {{ recordStatusMap[record.id].movement.type }} × {{ recordStatusMap[record.id].movement.quantity }}
                                            </span>
                                        </div>
                                        <div
                                            v-else-if="recordStatusMap[record.id]?.reason"
                                            class="mt-1 text-[11px] text-fg-mid-grey italic"
                                        >
                                            Note: {{ recordStatusMap[record.id].reason }}
                                        </div>
                                    </td>

                                    <!-- Import Status -->
                                    <td class="px-2 py-2.5">
                                        <span
                                            v-if="recordStatusMap[record.id]?.status === 'imported'"
                                            class="inline-flex items-center gap-1 rounded-full bg-fg-positive-15 px-2 py-0.5 text-[11px] font-semibold text-fg-positive-dark"
                                        >
                                            <span>●</span> Imported
                                        </span>
                                        <span
                                            v-else-if="recordStatusMap[record.id]?.status === 'analyzing'"
                                            class="inline-flex items-center gap-1 rounded-full bg-fg-main-blue-15 px-2 py-0.5 text-[11px] font-semibold text-fg-main-blue animate-pulse"
                                        >
                                            <span>◌</span> Analyzing…
                                        </span>
                                        <span
                                            v-else-if="recordStatusMap[record.id]?.status === 'skipped'"
                                            class="inline-flex items-center gap-1 rounded-full bg-fg-pale-grey px-2 py-0.5 text-[11px] font-medium text-fg-mid-grey"
                                        >
                                            <span>—</span> Skipped
                                        </span>
                                        <span
                                            v-else
                                            class="inline-flex items-center gap-1 rounded-full bg-fg-warning-15 px-2 py-0.5 text-[11px] font-medium text-fg-warning-text"
                                        >
                                            <span>○</span> Pending
                                        </span>
                                    </td>

                                    <!-- Quick Actions -->
                                    <td class="whitespace-nowrap px-3 py-2.5 text-right">
                                        <div class="flex items-center justify-end gap-1">
                                            <!-- AI Quick Action -->
                                            <button
                                                v-if="recordStatusMap[record.id]?.status === 'pending'"
                                                class="rounded border border-fg-main-blue-30 bg-white px-2 py-1 text-[11px] font-semibold text-fg-main-blue shadow-2xs hover:bg-fg-main-blue hover:text-white transition disabled:opacity-50 cursor-pointer"
                                                title="Use Claude AI to analyze and import this record"
                                                :disabled="aiLoading"
                                                @click="analyzeSingleRecord(record)"
                                            >
                                                ✨ AI
                                            </button>

                                            <!-- Fill manual form button -->
                                            <button
                                                class="rounded border border-fg-muted-grey bg-white px-2 py-1 text-[11px] font-medium text-fg-mid-grey shadow-2xs hover:border-fg-dark-blue hover:text-fg-dark-blue transition cursor-pointer"
                                                title="Prefill left-side manual form"
                                                @click="prefillRecordToForm(record)"
                                            >
                                                Fill Form
                                            </button>
                                        </div>
                                    </td>
                                </tr>

                                <tr v-if="filteredRecords.length === 0">
                                    <td colspan="7" class="py-8 text-center text-fg-light-grey">
                                        No records found in this view.
                                    </td>
                                </tr>
                            </tbody>
                        </table>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>
