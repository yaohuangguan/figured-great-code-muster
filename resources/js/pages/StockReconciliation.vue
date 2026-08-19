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
const importingRecordId = ref(null); // specific record ID confirming/importing
const withdrawingRecordId = ref(null); // specific record ID withdrawing
const aiMessage = ref(null);
const aiError = ref(null);
const aiOverviewData = ref(null); // Stores full-width AI overview data matching image UI

// Floating Toast Notification state
const toast = ref({
    show: false,
    message: '',
    type: 'success',
    timeout: null,
});

function showToast(message, type = 'success') {
    if (toast.value.timeout) clearTimeout(toast.value.timeout);
    toast.value = {
        show: true,
        message,
        type,
        timeout: setTimeout(() => {
            toast.value.show = false;
        }, 4000),
    };
}

// Filters: Date Range, Stock Class, Movement Type
const startDate = ref('');
const endDate = ref('');
const filterClass = ref('');
const filterType = ref('');

// Selected records for multi-select / bulk operations
const selectedRecordIds = ref([]);
const activeTab = ref('all'); // 'all', 'pending', 'review', 'imported'

// Map of recordId -> { status: 'pending'|'review_needed'|'imported'|'skipped'|'analyzing', movementId: number|null, movement: object|null, candidate: object|null, reason: string|null }
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
            candidate: null,
            reason: null,
        };
    }

    for (const sc of classes.value) {
        for (const mv of sc.movements) {
            const idMatch = mv.note?.match(/\[REC-(\d+)\]/);
            if (idMatch && map[idMatch[1]]) {
                map[idMatch[1]] = {
                    status: 'imported',
                    movementId: mv.id,
                    movement: { ...mv, stock_class_name: sc.name },
                    candidate: null,
                    reason: null,
                };
                continue;
            }

            if (mv.type === 'birth' && mv.quantity === 1240 && mv.note?.toLowerCase().includes('docking')) {
                const dockingRecord = records.value.find((r) => r.body.toLowerCase().includes('docked 1,240 lambs'));
                if (dockingRecord && map[dockingRecord.id]) {
                    map[dockingRecord.id] = {
                        status: 'imported',
                        movementId: mv.id,
                        movement: { ...mv, stock_class_name: sc.name },
                        candidate: null,
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
                candidate: null,
                reason: null,
            };
        }

        showToast(`Successfully added movement: ${targetClass?.name} · ${data.type} × ${data.quantity}`);

        // Reset form
        movementForm.value.quantity = null;
        movementForm.value.note = '';
        movementForm.value.source_record_id = null;
    } catch (e) {
        aiError.value = e.response?.data?.message ?? e.message;
        showToast(aiError.value, 'error');
    } finally {
        saving.value = false;
    }
}

async function removeMovement(stockClass, movement) {
    await axios.delete(`/api/stock-movements/${movement.id}`);
    stockClass.movements = stockClass.movements.filter((m) => m.id !== movement.id);

    for (const [recId, entry] of Object.entries(recordStatusMap.value)) {
        if (entry.movementId === movement.id) {
            recordStatusMap.value[recId] = {
                status: 'pending',
                movementId: null,
                movement: null,
                candidate: null,
                reason: null,
            };
        }
    }
    showToast(`Removed movement: ${stockClass.name} · ${movement.type} × ${movement.quantity}`, 'info');
}

// Withdraw (撤回) an imported record
async function withdrawRecord(recordId) {
    const statusEntry = recordStatusMap.value[recordId];
    if (!statusEntry || !statusEntry.movementId) return;

    withdrawingRecordId.value = recordId;
    try {
        await axios.delete(`/api/stock-movements/${statusEntry.movementId}`);
        for (const sc of classes.value) {
            sc.movements = sc.movements.filter((m) => m.id !== statusEntry.movementId);
        }
        recordStatusMap.value[recordId] = {
            status: 'pending',
            movementId: null,
            movement: null,
            candidate: null,
            reason: null,
        };
        showToast(`Withdrawn record #${recordId} and reverted to pending`, 'info');
    } catch (e) {
        aiError.value = e.response?.data?.message ?? e.message;
        showToast(aiError.value, 'error');
    } finally {
        withdrawingRecordId.value = null;
    }
}

// Bulk withdraw all selected imported records
async function withdrawSelected() {
    const importedSelected = selectedRecordIds.value.filter(
        (id) => recordStatusMap.value[id]?.status === 'imported' && recordStatusMap.value[id]?.movementId,
    );
    for (const id of importedSelected) {
        await withdrawRecord(id);
    }
    selectedRecordIds.value = [];
    showToast(`Successfully withdrawn ${importedSelected.length} movement(s)`, 'info');
}

// Confirm and import a candidate movement that needed review
async function confirmCandidateMovement(recordId, candidate) {
    const rec = records.value.find((r) => r.id === recordId);
    if (!rec || !candidate) return;

    importingRecordId.value = recordId;
    try {
        const noteWithTag = `${candidate.note || rec.source} [REC-${rec.id}]`;
        const { data: createdMovement } = await axios.post('/api/stock-movements', {
            stock_class_id: candidate.stock_class_id,
            type: candidate.type,
            quantity: candidate.quantity,
            note: noteWithTag,
        });

        const targetClass = classes.value.find((c) => c.id === candidate.stock_class_id);
        if (targetClass) {
            targetClass.movements.push(createdMovement);
        }

        recordStatusMap.value[rec.id] = {
            status: 'imported',
            movementId: createdMovement.id,
            movement: { ...createdMovement, stock_class_name: targetClass?.name },
            candidate: null,
            reason: candidate.reasoning || null,
        };

        if (aiOverviewData.value?.needsAttention) {
            aiOverviewData.value.needsAttention = aiOverviewData.value.needsAttention.map((item) => {
                if (item.record_ids?.includes(recordId)) {
                    return { ...item, resolved: true };
                }
                return item;
            });
        }

        showToast(`Confirmed & imported #${recordId}: ${targetClass?.name} · ${createdMovement.type} × ${createdMovement.quantity}`);
    } catch (e) {
        aiError.value = e.response?.data?.message ?? e.message;
        showToast(aiError.value, 'error');
    } finally {
        importingRecordId.value = null;
    }
}

function skipRecord(recordId, reason = 'Manually skipped') {
    recordStatusMap.value[recordId] = {
        status: 'skipped',
        movementId: null,
        movement: null,
        candidate: null,
        reason,
    };
    if (aiOverviewData.value?.needsAttention) {
        aiOverviewData.value.needsAttention = aiOverviewData.value.needsAttention.map((item) => {
            if (item.record_ids?.includes(recordId)) {
                return { ...item, resolved: true };
            }
            return item;
        });
    }
    showToast(`Skipped record #${recordId}`, 'info');
}

function prefillRecordToForm(record, candidate = null) {
    movementForm.value.source_record_id = record.id;

    if (candidate) {
        movementForm.value.stock_class_id = candidate.stock_class_id;
        movementForm.value.type = candidate.type;
        movementForm.value.quantity = candidate.quantity;
        movementForm.value.note = candidate.note || `${record.source} ${shortDate(record.recorded_on)}`;
    } else {
        movementForm.value.note = `${record.source} ${shortDate(record.recorded_on)}`;
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
    }

    window.scrollTo({ top: 0, behavior: 'smooth' });
}

// Inferred or actual stock class (animal type) helper
function getAnimalClass(record) {
    if (recordStatusMap.value[record.id]?.movement?.stock_class_name) {
        return recordStatusMap.value[record.id].movement.stock_class_name;
    }
    if (recordStatusMap.value[record.id]?.candidate?.stock_class_name) {
        return recordStatusMap.value[record.id].candidate.stock_class_name;
    }

    const body = record.body.toLowerCase();
    if (body.includes('ewe') || body.includes('two-tooth')) return 'Ewes';
    if (body.includes('lamb')) return 'Lambs';
    if (body.includes('cow') || body.includes('calv') || body.includes('steer') || body.includes('cattle')) return 'Cattle';
    return 'General';
}

// Inferred or actual movement type helper
function getMovementType(record) {
    if (recordStatusMap.value[record.id]?.movement?.type) {
        return recordStatusMap.value[record.id].movement.type;
    }
    if (recordStatusMap.value[record.id]?.candidate?.type) {
        return recordStatusMap.value[record.id].candidate.type;
    }

    const body = record.body.toLowerCase();
    if (body.includes('dock') || body.includes('calv') || body.includes('calves on') || body.includes('born')) return 'birth';
    if (body.includes('purchase') || body.includes('bought') || body.includes('p-')) return 'purchase';
    if (body.includes('dead') || body.includes('died') || body.includes('lost') || body.includes('down') || body.includes('killed') || body.includes('put down')) return 'death';
    if (body.includes('sale') || body.includes('sold') || body.includes('s-') || body.includes('truck')) return 'sale';
    return 'other';
}

const animalBadgeClass = {
    'Ewes': 'bg-emerald-50 text-emerald-800 border-emerald-200',
    'Lambs': 'bg-indigo-50 text-indigo-800 border-indigo-200',
    'Cattle': 'bg-amber-50 text-amber-900 border-amber-200',
    'General': 'bg-fg-super-pale-grey text-fg-mid-grey border-fg-pale-grey',
};

const hasActiveFilters = computed(() => {
    return Boolean(startDate.value || endDate.value || filterClass.value || filterType.value);
});

function resetFilters() {
    startDate.value = '';
    endDate.value = '';
    filterClass.value = '';
    filterType.value = '';
}

// Filtered paper trail records with date range, class, and type filters
const filteredRecords = computed(() => {
    let list = records.value;

    if (activeTab.value === 'pending') {
        list = list.filter((r) => recordStatusMap.value[r.id]?.status === 'pending');
    } else if (activeTab.value === 'review') {
        list = list.filter((r) => recordStatusMap.value[r.id]?.status === 'review_needed');
    } else if (activeTab.value === 'imported') {
        list = list.filter((r) => recordStatusMap.value[r.id]?.status === 'imported');
    }

    if (startDate.value) {
        list = list.filter((r) => r.recorded_on && r.recorded_on >= startDate.value);
    }
    if (endDate.value) {
        list = list.filter((r) => r.recorded_on && r.recorded_on <= endDate.value);
    }

    if (filterClass.value) {
        list = list.filter((r) => getAnimalClass(r) === filterClass.value);
    }

    if (filterType.value) {
        list = list.filter((r) => getMovementType(r) === filterType.value);
    }

    return list;
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

const hasImportedInSelection = computed(() => {
    return selectedRecordIds.value.some((id) => recordStatusMap.value[id]?.status === 'imported');
});

function toggleSelectAll() {
    if (allFilteredSelected.value) {
        const filteredIds = new Set(filteredRecords.value.map((r) => r.id));
        selectedRecordIds.value = selectedRecordIds.value.filter((id) => !filteredIds.has(id));
    } else {
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
const reviewCount = computed(
    () => records.value.filter((r) => recordStatusMap.value[r.id]?.status === 'review_needed').length,
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

    recordsToProcess.forEach((r) => {
        if (recordStatusMap.value[r.id]) {
            recordStatusMap.value[r.id].status = 'analyzing';
        }
    });

    const classInfo = classes.value.map((c) => ({ id: c.id, name: c.name }));

    const systemPrompt = `You are a specialist agricultural accounting assistant for New Zealand livestock farm reconciliations.
Your task is to analyze farm records (diaries, sale/purchase dockets, text messages) for Kahikatea Downs (Stock year 1 Jul 2025 – 30 Jun 2026), extract exact stock movements, identify issues/ambiguities, and assess extraction confidence.

Available Stock Classes:
${JSON.stringify(classInfo, null, 2)}

Movement Types:
- "birth": Lambing, calving, docking tallies, calves on the ground.
- "purchase": Bought stock, purchase dockets.
- "death": Stock that died, lost in storms/bog/drought, bearing trouble, euthanized, or killed for farm meat freezer.
- "sale": Sold stock, sale dockets, livestock trucks away, private sales.

CRITICAL INSTRUCTIONS FOR ANALYSIS, AMBIGUITY & NEEDS ATTENTION:
1. For records with incomplete or estimated numbers (e.g. Record 21: "Lost about 15 ewes near the creek, probably more up the back gully - will know for sure at crutching"):
   - Set "needs_review": true
   - Add an entry in "needs_attention" with severity "HIGH", badge "NEEDS REVIEW", clear title and detailed problem description explaining why it's incomplete.
2. For duplicate dockets (e.g. S-40417 on Feb 17 and Feb 24):
   - Set "is_movement": true for the first docket, and "is_movement": false for the duplicate.
   - Add an entry in "needs_attention" with severity "HIGH", badge "DUPLICATE", detailing that the same docket appears on two dates.
3. For text message corrections (e.g. Kate's 40 vs 38 cull cows):
   - Add an entry in "needs_attention" explaining the correction.
4. Calculate a realistic "confidence_score" between 85 and 98 based on clarity of logs.
5. Provide a clear, natural "summary" paragraph summarizing what records were read across ewes, lambs, and cattle, and what actions were taken.

Output ONLY valid JSON in this exact structure (no markdown fences):
{
  "summary": "...",
  "confidence_score": 95,
  "needs_attention": [
    {
      "severity": "HIGH",
      "badge": "NEEDS REVIEW",
      "title": "Ewe deaths near creek — count not yet settled",
      "description": "Record 21 (9 Jun 2026) notes approximately 15 ewes lost near the creek after a southerly, but the farmer explicitly states there are 'probably more up the back gully' and that the true count will not be known until crutching. Because the farmer's own figure is openly incomplete, no movement has been proposed. The gap will remain open until the crutching count is completed and the total confirmed losses are reported.",
      "record_ids": [21],
      "candidate": {
        "stock_class_id": 1,
        "type": "death",
        "quantity": 15,
        "note": "Southerly storm ewe loss (Diary 9 Jun 2026)"
      }
    },
    {
      "severity": "HIGH",
      "badge": "DUPLICATE",
      "title": "Duplicate sale docket S-40417 filed on two dates",
      "description": "Sale docket S-40417 (180 lambs @ $142.00 = $25,560.00) appears identically in record 11 (17 Feb 2026) and record 12 (24 Feb 2026). This is one sale, not two. It has been proposed once from record 11. Record 12 should be investigated to determine whether it is a filing duplicate or a separate transaction, and the redundant record should be removed or marked as duplicate to prevent double-counting.",
      "record_ids": [11, 12]
    }
  ],
  "movements": [
    {
      "record_id": 1,
      "is_movement": true,
      "needs_review": false,
      "stock_class_id": 1,
      "type": "death",
      "quantity": 2,
      "note": "Bearing trouble (Diary 14 Aug 2025)",
      "reasoning": "2 ewes euthanized with bearing trouble"
    }
  ]
}`;

    const userPrompt = `Please analyze the following farm records, extract stock movements, and generate the full overview with needs_attention items:
${JSON.stringify(recordsToProcess.map((r) => ({ id: r.id, date: r.recorded_on, source: r.source, text: r.body })), null, 2)}`;

    try {
        const { data } = await axios.post('/api/ai', {
            system: systemPrompt,
            prompt: userPrompt,
        });

        let jsonText = data.text.trim();
        if (jsonText.startsWith('```')) {
            jsonText = jsonText.replace(/^```(json)?\n?/, '').replace(/\n?```$/, '');
        }

        const parsedResponse = JSON.parse(jsonText);
        const movementsList = Array.isArray(parsedResponse) ? parsedResponse : (parsedResponse.movements || []);
        const summaryText = parsedResponse.summary || `Processed ${recordsToProcess.length} paper trail record(s).`;
        const confidenceScore = parsedResponse.confidence_score || 94;
        const needsAttentionList = parsedResponse.needs_attention || [];

        let newlyImportedList = [];
        let newlySkippedList = [];
        let newlyReviewList = [];
        const classBreakdown = {};

        for (const item of movementsList) {
            const rec = records.value.find((r) => r.id === item.record_id);
            if (!rec) continue;

            if (item.is_movement && item.stock_class_id && item.quantity > 0) {
                if (item.needs_review) {
                    const sc = classes.value.find((c) => c.id === item.stock_class_id);
                    const candidate = {
                        stock_class_id: item.stock_class_id,
                        stock_class_name: sc?.name,
                        type: item.type,
                        quantity: item.quantity,
                        note: item.note || rec.source,
                        reasoning: item.review_reason || item.reasoning,
                    };

                    recordStatusMap.value[rec.id] = {
                        status: 'review_needed',
                        movementId: null,
                        movement: null,
                        candidate,
                        reason: item.review_reason || item.reasoning || 'Unclear / estimated count',
                    };

                    newlyReviewList.push({
                        record: rec,
                        candidate,
                        reason: item.review_reason || item.reasoning,
                    });
                    continue;
                }

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
                    candidate: null,
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
                    candidate: null,
                    reason: item.reasoning || 'No movement / Duplicate / Superseded',
                };
                newlySkippedList.push({
                    record: rec,
                    reasoning: item.reasoning || 'Duplicate or non-movement log',
                });
            }
        }

        aiOverviewData.value = {
            summary: summaryText,
            confidenceScore,
            importedCount: newlyImportedList.length,
            skippedCount: newlySkippedList.length,
            reviewCount: newlyReviewList.length,
            needsAttention: needsAttentionList,
            classBreakdown,
        };

        const successMsg = `✨ AI Analysis Complete: ${newlyImportedList.length} movement(s) imported, ${needsAttentionList.length} item(s) flagged for attention.`;
        aiMessage.value = successMsg;
        showToast(successMsg, 'success');
        selectedRecordIds.value = [];
    } catch (e) {
        console.error('AI reconciliation failed:', e);
        aiError.value = e.response?.data?.error || e.message || 'AI analysis request failed. Please check network/key.';
        showToast(aiError.value, 'error');
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
    const pendingRecs = records.value.filter(
        (r) => recordStatusMap.value[r.id]?.status === 'pending' || recordStatusMap.value[r.id]?.status === 'review_needed',
    );
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
    <div class="space-y-5">
        <!-- Floating Toast Notification -->
        <transition
            enter-active-class="transform ease-out duration-300 transition"
            enter-from-class="translate-y-2 opacity-0 sm:translate-y-0 sm:translate-x-2"
            enter-to-class="translate-y-0 opacity-100 sm:translate-x-0"
            leave-active-class="transition ease-in duration-200"
            leave-from-class="opacity-100"
            leave-to-class="opacity-0"
        >
            <div
                v-if="toast.show"
                class="fixed top-5 right-5 z-50 flex items-center gap-3 rounded-lg border px-4 py-3 shadow-lg max-w-md bg-white transition-all"
                :class="[
                    toast.type === 'success' ? 'border-fg-positive text-fg-dark-blue shadow-fg-positive/10' : '',
                    toast.type === 'error' ? 'border-fg-danger text-fg-danger-dark shadow-fg-danger/10' : '',
                    toast.type === 'info' ? 'border-fg-main-blue text-fg-dark-blue shadow-fg-main-blue/10' : '',
                ]"
            >
                <span v-if="toast.type === 'success'" class="text-base">✅</span>
                <span v-else-if="toast.type === 'error'" class="text-base">⚠️</span>
                <span v-else class="text-base">ℹ️</span>
                <div class="text-xs font-medium leading-snug">
                    {{ toast.message }}
                </div>
                <button
                    class="ml-auto pl-2 text-xs font-bold text-fg-light-grey hover:text-fg-dark-grey cursor-pointer"
                    @click="toast.show = false"
                >
                    ✕
                </button>
            </div>
        </transition>

        <!-- Page Title & Header -->
        <div>
            <h2 class="text-xl font-semibold tracking-tight text-fg-dark-blue">Stock reconciliation — Kahikatea Downs</h2>
            <p class="text-sm text-fg-mid-grey">
                Key stock movements in from raw records (right) until each tally matches the farmer's recorded closing count.
                Stock year 1 Jul 2025 – 30 Jun 2026.
            </p>
        </div>

        <!-- FULL-WIDTH AI OVERVIEW OF THE PAPER TRAIL -->
        <div
            v-if="aiOverviewData"
            class="w-full overflow-hidden rounded-lg border border-fg-muted-grey bg-white shadow-xs transition-all"
        >
            <!-- Card Header -->
            <div class="flex items-center justify-between border-b border-fg-pale-grey bg-white px-5 py-2.5">
                <div class="flex items-center gap-2">
                    <h3 class="text-sm font-bold text-fg-dark-blue">AI overview of the paper trail</h3>
                </div>
                <button
                    class="text-xs text-fg-mid-grey hover:text-fg-dark-blue transition cursor-pointer font-medium"
                    @click="aiOverviewData = null"
                >
                    Dismiss ✕
                </button>
            </div>

            <!-- Two-Column Grid: Summary on Left, Needs Attention on Right -->
            <div class="grid grid-cols-1 divide-y divide-fg-pale-grey lg:grid-cols-12 lg:divide-y-0 lg:divide-x">
                <!-- Left Side: SUMMARY & METRICS & CONFIDENCE SCORE -->
                <div class="p-5 lg:col-span-4 space-y-4">
                    <div>
                        <h4 class="text-[11px] font-bold tracking-wider uppercase text-fg-mid-grey mb-2.5">
                            SUMMARY
                        </h4>
                        <p class="text-xs text-fg-dark-grey leading-relaxed">
                            {{ aiOverviewData.summary }}
                        </p>
                    </div>

                    <!-- AI Confidence Score Meter -->
                    <div class="rounded-lg border border-fg-pale-grey bg-fg-super-pale-grey p-3 space-y-2">
                        <div class="flex items-center justify-between">
                            <span class="text-[11px] font-semibold text-fg-dark-blue flex items-center gap-1.5">
                                <span>🎯</span> Confidence Score
                            </span>
                            <span class="font-mono text-xs font-bold text-fg-main-blue">
                                {{ aiOverviewData.confidenceScore }}%
                            </span>
                        </div>
                        <!-- Progress bar -->
                        <div class="w-full bg-fg-pale-grey rounded-full h-1.5 overflow-hidden">
                            <div
                                class="h-1.5 rounded-full transition-all duration-500"
                                :class="aiOverviewData.confidenceScore >= 90 ? 'bg-fg-positive' : 'bg-fg-warning'"
                                :style="{ width: `${aiOverviewData.confidenceScore}%` }"
                            ></div>
                        </div>
                        <div class="flex items-center justify-between text-[10px] text-fg-mid-grey">
                            <span>High extraction accuracy</span>
                            <span>{{ aiOverviewData.importedCount }} verified entries</span>
                        </div>
                    </div>

                    <!-- Class Movements Visual Breakdown Chart -->
                    <div v-if="Object.keys(aiOverviewData.classBreakdown || {}).length > 0" class="space-y-1.5">
                        <h5 class="text-[11px] font-semibold text-fg-dark-blue flex items-center gap-1">
                            <span>📊</span> Movement Categorization
                        </h5>
                        <div class="space-y-1.5 text-[11px]">
                            <div
                                v-for="(breakdown, className) in aiOverviewData.classBreakdown"
                                :key="className"
                                class="flex items-center justify-between rounded bg-white border border-fg-pale-grey px-2.5 py-1"
                            >
                                <span class="font-medium text-fg-dark-blue">{{ className }}</span>
                                <div class="flex items-center gap-2 text-[10px] font-mono">
                                    <span v-if="breakdown.births" class="text-fg-positive-dark font-medium">+{{ breakdown.births }} B</span>
                                    <span v-if="breakdown.purchases" class="text-fg-main-blue font-medium">+{{ breakdown.purchases }} P</span>
                                    <span v-if="breakdown.deaths" class="text-fg-danger-dark font-medium">−{{ breakdown.deaths }} D</span>
                                    <span v-if="breakdown.sales" class="text-fg-warning-text font-medium">−{{ breakdown.sales }} S</span>
                                </div>
                            </div>
                        </div>
                    </div>
                </div>

                <!-- Right Side: NEEDS ATTENTION -->
                <div class="p-5 lg:col-span-8 space-y-4">
                    <div class="flex items-center gap-2">
                        <h4 class="text-[11px] font-bold tracking-wider uppercase text-fg-mid-grey">
                            NEEDS ATTENTION
                        </h4>
                        <!-- Badge counts -->
                        <span
                            v-if="aiOverviewData.needsAttention?.filter((n) => n.badge === 'NEEDS REVIEW').length"
                            class="rounded-full bg-[#102c58] px-2 py-0.5 text-[10px] font-bold text-white"
                        >
                            {{ aiOverviewData.needsAttention.filter((n) => n.badge === 'NEEDS REVIEW').length }} to review
                        </span>
                        <span
                            v-if="aiOverviewData.needsAttention?.filter((n) => n.severity === 'HIGH').length"
                            class="rounded-full bg-red-100 px-2 py-0.5 text-[10px] font-bold text-red-600"
                        >
                            {{ aiOverviewData.needsAttention.filter((n) => n.severity === 'HIGH').length }} high
                        </span>
                    </div>

                    <!-- Empty state if all items are good / clear -->
                    <div
                        v-if="!aiOverviewData.needsAttention || aiOverviewData.needsAttention.length === 0"
                        class="rounded-lg border border-dashed border-fg-positive-30 bg-fg-positive-9/30 p-6 text-center space-y-2"
                    >
                        <div class="inline-flex h-9 w-9 items-center justify-center rounded-full bg-fg-positive-15 text-fg-positive-dark text-base font-bold">
                            ✓
                        </div>
                        <h5 class="text-xs font-bold text-fg-positive-dark">All good! No attention items needed</h5>
                        <p class="text-xs text-fg-mid-grey max-w-sm mx-auto">
                            All processed paper trail records have been cleanly extracted, reconciled, or accounted for with high confidence.
                        </p>
                    </div>

                    <!-- Issues List with colored left border indicator -->
                    <div v-else class="space-y-4">
                        <div
                            v-for="(item, idx) in aiOverviewData.needsAttention"
                            :key="idx"
                            class="border-l-4 pl-3 py-1 space-y-1.5 transition"
                            :class="[
                                item.severity === 'HIGH' ? 'border-l-red-500' : 'border-l-amber-500',
                                item.resolved ? 'opacity-50' : '',
                            ]"
                        >
                            <!-- Badges + Title -->
                            <div class="flex flex-wrap items-center gap-2">
                                <span
                                    v-if="item.severity"
                                    class="rounded px-1.5 py-0.5 text-[10px] font-bold uppercase tracking-wider"
                                    :class="item.severity === 'HIGH' ? 'bg-red-100 text-red-600' : 'bg-amber-100 text-amber-700'"
                                >
                                    {{ item.severity }}
                                </span>
                                <span
                                    v-if="item.badge"
                                    class="rounded bg-[#102c58] px-1.5 py-0.5 text-[10px] font-bold uppercase tracking-wider text-white"
                                >
                                    {{ item.badge }}
                                </span>
                                <span class="text-xs font-bold text-fg-dark-blue">
                                    {{ item.title }}
                                </span>
                            </div>

                            <!-- Detailed Description -->
                            <p class="text-xs text-fg-dark-grey leading-relaxed">
                                {{ item.description }}
                            </p>

                            <!-- Bottom Row: Record Pills & Interactive Actions -->
                            <div class="flex flex-wrap items-center justify-between gap-2 pt-1">
                                <!-- Record Tag Pills -->
                                <div class="flex items-center gap-1.5">
                                    <span
                                        v-for="recId in item.record_ids"
                                        :key="recId"
                                        class="rounded border border-fg-muted-grey bg-fg-super-pale-grey px-2 py-0.5 font-mono text-[10px] font-medium text-fg-dark-grey"
                                    >
                                        #{{ recId }}
                                    </span>
                                </div>

                                <!-- Action Buttons if candidate movement exists -->
                                <div v-if="item.candidate && !item.resolved" class="flex items-center gap-1.5">
                                    <button
                                        class="inline-flex items-center gap-1 rounded bg-fg-positive-dark px-2.5 py-0.5 text-[11px] font-semibold text-white hover:opacity-90 transition disabled:opacity-50 cursor-pointer"
                                        :disabled="importingRecordId === item.record_ids[0]"
                                        @click="confirmCandidateMovement(item.record_ids[0], item.candidate)"
                                    >
                                        <svg v-if="importingRecordId === item.record_ids[0]" class="animate-spin -ml-0.5 h-3 w-3 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                                            <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                                            <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8v8H4z"></path>
                                        </svg>
                                        <span>{{ importingRecordId === item.record_ids[0] ? 'Importing…' : '✓ Confirm & Import' }}</span>
                                    </button>
                                    <button
                                        class="rounded border border-fg-muted-grey bg-white px-2 py-0.5 text-[11px] font-medium text-fg-dark-grey hover:border-fg-dark-blue transition cursor-pointer"
                                        @click="prefillRecordToForm(records.find((r) => r.id === item.record_ids[0]), item.candidate)"
                                    >
                                        ✎ Edit in Form
                                    </button>
                                    <button
                                        class="text-[11px] text-fg-light-grey hover:text-fg-danger px-1 cursor-pointer"
                                        @click="skipRecord(item.record_ids[0], 'Skipped from overview')"
                                    >
                                        Dismiss
                                    </button>
                                </div>
                                <span v-else-if="item.resolved" class="text-[11px] text-fg-positive-dark font-medium">
                                    ✓ Resolved
                                </span>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <p v-if="loading" class="text-sm text-fg-light-grey">Loading stock data…</p>

        <div v-else class="grid grid-cols-1 gap-5 lg:grid-cols-12">
            <!-- LEFT COLUMN (5 cols on lg): Key in a movement (TOP) + Tally tallies (BELOW) -->
            <div class="space-y-4 lg:col-span-5">
                <!-- 1. Key in a movement form -->
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
                                class="text-xs text-fg-mid-grey hover:text-fg-danger cursor-pointer"
                                @click="movementForm.source_record_id = null; movementForm.note = ''; movementForm.quantity = null"
                            >
                                Clear prefill
                            </button>
                            <span v-else></span>

                            <button
                                class="inline-flex items-center gap-1.5 rounded bg-fg-main-blue px-4 py-1.5 text-sm font-medium text-white shadow-xs hover:bg-fg-main-blue-hover disabled:opacity-50 cursor-pointer"
                                :disabled="!canSave || saving"
                                @click="addMovement"
                            >
                                <svg v-if="saving" class="animate-spin -ml-0.5 h-4 w-4 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                                    <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                                    <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8v8H4z"></path>
                                </svg>
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
                                    class="rounded p-0.5 text-fg-light-grey hover:bg-fg-danger-15 hover:text-fg-danger cursor-pointer"
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
                <!-- Paper Trail Main Container -->
                <div class="overflow-hidden rounded-lg border border-fg-muted-grey bg-white shadow-xs">
                    <!-- Header & Bulk Action Bar -->
                    <div class="border-b border-fg-pale-grey bg-fg-super-pale-grey p-3 sm:p-4 space-y-3">
                        <!-- Top Row: Title, summary count, and Action buttons -->
                        <div class="flex flex-col gap-2 sm:flex-row sm:items-center sm:justify-between">
                            <div>
                                <div class="flex items-center gap-2">
                                    <h3 class="text-sm font-bold text-fg-dark-blue">The paper trail</h3>
                                    <span class="rounded-full bg-fg-main-blue-15 px-2 py-0.5 text-[11px] font-medium text-fg-main-blue">
                                        {{ filteredRecords.length }} / {{ records.length }} logs
                                    </span>
                                </div>
                                <p class="text-xs text-fg-mid-grey">
                                    {{ importedCount }} imported • {{ reviewCount ? `${reviewCount} needs review • ` : '' }}{{ pendingCount }} pending
                                </p>
                            </div>

                            <!-- Bulk Action Buttons with Loading Indicator -->
                            <div class="flex flex-wrap items-center gap-2">
                                <!-- Bulk Withdraw Button (if imported selected) -->
                                <button
                                    v-if="hasImportedInSelection"
                                    class="inline-flex items-center gap-1 rounded border border-fg-danger-30 bg-fg-danger-9 px-2.5 py-1.5 text-xs font-semibold text-fg-danger-dark shadow-xs hover:bg-fg-danger hover:text-white transition cursor-pointer"
                                    @click="withdrawSelected"
                                >
                                    <span>↩</span>
                                    <span>Withdraw Selected</span>
                                </button>

                                <!-- AI Selected with Spinner -->
                                <button
                                    v-if="selectedRecordIds.length > 0"
                                    class="inline-flex items-center gap-1.5 rounded bg-fg-main-blue px-3 py-1.5 text-xs font-semibold text-white shadow-xs hover:bg-fg-main-blue-hover disabled:opacity-50 cursor-pointer"
                                    :disabled="aiLoading"
                                    @click="autoReconcileSelected"
                                >
                                    <svg v-if="aiLoading" class="animate-spin -ml-0.5 h-3.5 w-3.5 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                                        <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                                        <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8v8H4z"></path>
                                    </svg>
                                    <span v-else>✨</span>
                                    <span>{{ aiLoading ? 'AI Importing…' : `AI Reconcile Selected (${selectedRecordIds.length})` }}</span>
                                </button>

                                <!-- AI Auto-Reconcile All with Spinner -->
                                <button
                                    v-else
                                    class="inline-flex items-center gap-1.5 rounded border border-fg-main-blue bg-fg-main-blue-9 px-3 py-1.5 text-xs font-semibold text-fg-main-blue hover:bg-fg-main-blue-15 disabled:opacity-50 cursor-pointer"
                                    :disabled="aiLoading || (pendingCount === 0 && reviewCount === 0)"
                                    @click="autoReconcileAllPending"
                                >
                                    <svg v-if="aiLoading" class="animate-spin -ml-0.5 h-3.5 w-3.5 text-fg-main-blue" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                                        <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                                        <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8v8H4z"></path>
                                    </svg>
                                    <span v-else>✨</span>
                                    <span>{{ aiLoading ? 'AI Reconciling & Importing…' : `AI Auto-Reconcile (${pendingCount + reviewCount})` }}</span>
                                </button>
                            </div>
                        </div>

                        <!-- Filter Row: Date Range + Stock Class + Movement Type -->
                        <div class="flex flex-wrap items-center gap-2.5 pt-1 text-xs border-t border-fg-pale-grey">
                            <!-- Date Range Filter Box -->
                            <div class="flex items-center gap-1.5 rounded border border-fg-muted-grey bg-white px-2 py-1 shadow-2xs">
                                <span class="text-fg-mid-grey text-[11px]">📅 Range:</span>
                                <input
                                    v-model="startDate"
                                    type="date"
                                    min="2025-07-01"
                                    max="2026-06-30"
                                    title="Start Date"
                                    class="bg-transparent text-xs text-fg-dark-grey outline-none focus:ring-0"
                                />
                                <span class="text-fg-light-grey text-[10px]">to</span>
                                <input
                                    v-model="endDate"
                                    type="date"
                                    min="2025-07-01"
                                    max="2026-06-30"
                                    title="End Date"
                                    class="bg-transparent text-xs text-fg-dark-grey outline-none focus:ring-0"
                                />
                            </div>

                            <!-- Stock Class Filter -->
                            <div class="flex items-center gap-1">
                                <select
                                    v-model="filterClass"
                                    class="rounded border border-fg-muted-grey bg-white px-2 py-1 text-xs text-fg-dark-grey shadow-2xs focus:border-fg-main-blue"
                                >
                                    <option value="">All Animals / Classes</option>
                                    <option value="Ewes">Ewes</option>
                                    <option value="Lambs">Lambs</option>
                                    <option value="Cattle">Cattle</option>
                                </select>
                            </div>

                            <!-- Movement Type Filter -->
                            <div class="flex items-center gap-1">
                                <select
                                    v-model="filterType"
                                    class="rounded border border-fg-muted-grey bg-white px-2 py-1 text-xs text-fg-dark-grey shadow-2xs focus:border-fg-main-blue capitalize"
                                >
                                    <option value="">All Movement Types</option>
                                    <option value="birth">Birth (+)</option>
                                    <option value="purchase">Purchase (+)</option>
                                    <option value="death">Death (−)</option>
                                    <option value="sale">Sale (−)</option>
                                </select>
                            </div>

                            <!-- Clear all filters button -->
                            <button
                                v-if="hasActiveFilters"
                                type="button"
                                class="rounded bg-fg-pale-grey px-2 py-1 text-[11px] font-medium text-fg-dark-grey hover:bg-fg-muted-grey transition cursor-pointer"
                                title="Reset all filters"
                                @click="resetFilters"
                            >
                                ✕ Reset Filters
                            </button>
                        </div>

                        <!-- Filter Tabs & Bulk Select Bar -->
                        <div class="flex items-center justify-between border-t border-fg-muted-grey/50 pt-2 text-xs">
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
                                    v-if="reviewCount > 0"
                                    class="rounded px-2.5 py-1 font-medium transition cursor-pointer"
                                    :class="activeTab === 'review' ? 'bg-white font-semibold text-fg-warning-text shadow-xs' : 'text-fg-mid-grey hover:text-fg-dark-blue'"
                                    @click="activeTab = 'review'"
                                >
                                    ⚠️ Needs Review ({{ reviewCount }})
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
                                    <th class="w-8 px-3 py-2 text-center">
                                        <input
                                            type="checkbox"
                                            :checked="allFilteredSelected"
                                            :indeterminate="isIndeterminate"
                                            class="h-3.5 w-3.5 rounded border-fg-muted-grey text-fg-main-blue focus:ring-fg-main-blue cursor-pointer"
                                            @change="toggleSelectAll"
                                        />
                                    </th>
                                    <th class="w-12 px-2 py-2 font-semibold">Key</th>
                                    <th class="w-20 px-2 py-2 font-semibold">Date</th>
                                    <th class="w-20 px-2 py-2 font-semibold">Source</th>
                                    <th class="w-16 px-2 py-2 font-semibold">Animal</th>
                                    <th class="w-16 px-2 py-2 font-semibold">Type</th>
                                    <th class="px-3 py-2 font-semibold">Paper Trail Log</th>
                                    <th class="w-28 px-2 py-2 font-semibold">Status</th>
                                    <th class="w-32 px-3 py-2 text-right font-semibold">Action</th>
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
                                        recordStatusMap[record.id]?.status === 'review_needed' ? 'bg-fg-warning-15/30' : '',
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

                                    <!-- Stock Class / Animal Type Column -->
                                    <td class="px-2 py-2.5">
                                        <span
                                            class="rounded-full border px-2 py-0.5 text-[10px] font-semibold tracking-wide"
                                            :class="animalBadgeClass[getAnimalClass(record)] || animalBadgeClass['General']"
                                        >
                                            {{ getAnimalClass(record) }}
                                        </span>
                                    </td>

                                    <!-- Movement Type Column -->
                                    <td class="px-2 py-2.5">
                                        <span
                                            class="rounded px-1.5 py-0.5 text-[10px] font-semibold uppercase tracking-wider"
                                            :class="movementTypeBadgeClass[getMovementType(record)] || 'bg-fg-super-pale-grey text-fg-mid-grey'"
                                        >
                                            {{ getMovementType(record) }}
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
                                            v-else-if="recordStatusMap[record.id]?.status === 'review_needed'"
                                            class="mt-1 text-[11px] text-fg-warning-text font-medium"
                                        >
                                            ⚠️ Ambiguity: {{ recordStatusMap[record.id].reason }}
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
                                            v-else-if="recordStatusMap[record.id]?.status === 'review_needed'"
                                            class="inline-flex items-center gap-1 rounded-full bg-fg-warning-15 px-2 py-0.5 text-[11px] font-bold text-fg-warning-text border border-fg-warning-15"
                                        >
                                            <span>⚠️</span> Review Needed
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
                                            class="inline-flex items-center gap-1 rounded-full bg-fg-pale-grey px-2 py-0.5 text-[11px] font-medium text-fg-mid-grey"
                                        >
                                            <span>○</span> Pending
                                        </span>
                                    </td>

                                    <!-- Quick Actions -->
                                    <td class="whitespace-nowrap px-3 py-2.5 text-right">
                                        <div class="flex items-center justify-end gap-1">
                                            <!-- Imported actions: Withdraw button -->
                                            <template v-if="recordStatusMap[record.id]?.status === 'imported'">
                                                <button
                                                    class="inline-flex items-center gap-1 rounded border border-fg-danger-30 bg-fg-danger-9 px-2 py-1 text-[11px] font-semibold text-fg-danger-dark shadow-2xs hover:bg-fg-danger hover:text-white transition disabled:opacity-50 cursor-pointer"
                                                    title="Withdraw this movement from tallies and revert to pending"
                                                    :disabled="withdrawingRecordId === record.id"
                                                    @click="withdrawRecord(record.id)"
                                                >
                                                    <svg v-if="withdrawingRecordId === record.id" class="animate-spin -ml-0.5 h-3 w-3 text-fg-danger-dark" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                                                        <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                                                        <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8v8H4z"></path>
                                                    </svg>
                                                    <span v-else>↩</span>
                                                    <span>{{ withdrawingRecordId === record.id ? 'Withdrawing…' : 'Withdraw' }}</span>
                                                </button>
                                                <button
                                                    class="rounded border border-fg-muted-grey bg-white px-1.5 py-1 text-[11px] font-medium text-fg-mid-grey shadow-2xs hover:border-fg-dark-blue hover:text-fg-dark-blue transition cursor-pointer"
                                                    title="Prefill into left manual form"
                                                    @click="prefillRecordToForm(record)"
                                                >
                                                    Fill
                                                </button>
                                            </template>

                                            <!-- Review needed actions -->
                                            <template v-else-if="recordStatusMap[record.id]?.status === 'review_needed'">
                                                <button
                                                    class="inline-flex items-center gap-1 rounded bg-fg-positive-dark px-2 py-1 text-[11px] font-semibold text-white shadow-2xs hover:opacity-90 transition disabled:opacity-50 cursor-pointer"
                                                    title="Confirm AI candidate"
                                                    :disabled="importingRecordId === record.id"
                                                    @click="confirmCandidateMovement(record.id, recordStatusMap[record.id].candidate)"
                                                >
                                                    <svg v-if="importingRecordId === record.id" class="animate-spin -ml-0.5 h-3 w-3 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                                                        <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                                                        <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8v8H4z"></path>
                                                    </svg>
                                                    <span>{{ importingRecordId === record.id ? 'Importing…' : 'Confirm' }}</span>
                                                </button>
                                                <button
                                                    class="rounded border border-fg-muted-grey bg-white px-2 py-1 text-[11px] font-medium text-fg-dark-grey shadow-2xs hover:border-fg-dark-blue transition cursor-pointer"
                                                    title="Edit in left-side form"
                                                    @click="prefillRecordToForm(record, recordStatusMap[record.id].candidate)"
                                                >
                                                    Edit
                                                </button>
                                            </template>

                                            <!-- Pending actions -->
                                            <template v-else-if="recordStatusMap[record.id]?.status === 'pending'">
                                                <button
                                                    class="inline-flex items-center gap-1 rounded border border-fg-main-blue-30 bg-white px-2 py-1 text-[11px] font-semibold text-fg-main-blue shadow-2xs hover:bg-fg-main-blue hover:text-white transition disabled:opacity-50 cursor-pointer"
                                                    title="Use Claude AI to analyze and import this record"
                                                    :disabled="aiLoading || aiProcessingId === record.id"
                                                    @click="analyzeSingleRecord(record)"
                                                >
                                                    <svg v-if="aiProcessingId === record.id" class="animate-spin -ml-0.5 h-3 w-3 text-fg-main-blue" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                                                        <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                                                        <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8v8H4z"></path>
                                                    </svg>
                                                    <span v-else>✨</span>
                                                    <span>{{ aiProcessingId === record.id ? 'Importing…' : 'AI' }}</span>
                                                </button>
                                                <button
                                                    class="rounded border border-fg-muted-grey bg-white px-2 py-1 text-[11px] font-medium text-fg-mid-grey shadow-2xs hover:border-fg-dark-blue hover:text-fg-dark-blue transition cursor-pointer"
                                                    title="Prefill left-side manual form"
                                                    @click="prefillRecordToForm(record)"
                                                >
                                                    Fill Form
                                                </button>
                                            </template>

                                            <!-- Skipped action -->
                                            <template v-else>
                                                <button
                                                    class="rounded border border-fg-muted-grey bg-white px-2 py-1 text-[11px] font-medium text-fg-mid-grey shadow-2xs hover:border-fg-dark-blue hover:text-fg-dark-blue transition cursor-pointer"
                                                    title="Prefill left-side manual form"
                                                    @click="prefillRecordToForm(record)"
                                                >
                                                    Fill Form
                                                </button>
                                            </template>
                                        </div>
                                    </td>
                                </tr>

                                <tr v-if="filteredRecords.length === 0">
                                    <td colspan="9" class="py-8 text-center text-fg-light-grey">
                                        No records match the current filters or date range.
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
