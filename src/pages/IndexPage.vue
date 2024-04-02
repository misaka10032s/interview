<template>
    <q-page class="row q-pt-xl">
        <div class="full-width q-px-xl">
            <div class="q-mb-xl">
                <q-input v-model="tempData.name" label="姓名" :rules="[val => !!val || '請輸入姓名']" />
                <q-input v-model="tempData.age" label="年齡" :rules="[val => !!val || '請輸入年齡', val => !isNaN(Number(val)) || '請輸入數字', val => Number(val) >= 0 || '請輸入正確年齡', val => Number.isInteger(Number(val)) || '請輸入整數']" type="number" />
                <q-btn color="primary" class="q-mt-md" @click="createData">{{ comfirmBtnText }}</q-btn>
                <q-btn color="grey-6" class="q-ml-md q-mt-md" v-if="editRow >= 0" @click="cancelEditData">取消</q-btn>
            </div>

            <div class="q-mb-xl">
                <q-btn @click="getData">get</q-btn>
                <q-btn @click="postData">post</q-btn>
                <q-btn @click="patchData">patch</q-btn>
                <q-btn @click="deleteData">delete</q-btn>
            </div>

            <!-- 7. 表格排序和篩選功能  -->
            <div class="q-mb-xl">
                <q-btn :color="filterAge == 50 ? 'primary' : 'secondary'" @click="filterAge = filterAge == 50 ? Infinity : 50">小於50歲</q-btn>
            </div>

            <q-table flat bordered ref="tableRef" :rows="filteredData" :columns="(tableConfig as QTableProps['columns'])" row-key="id" hide-pagination separator="cell" :rows-per-page-options="[0]">
                <template v-slot:header="props">
                    <q-tr :props="props">
                        <q-th v-for="col in props.cols" :key="col.name" :props="props" @click="sort(col.field)" style="cursor: pointer;">
                            {{ col.label }}
                            <q-icon v-if="sortBy.key === col.field" :name="`arrow_${sortBy.order === 'asc' ? 'upward' : 'downward'}`" />
                        </q-th>
                        <q-th></q-th>
                    </q-tr>
                </template>

                <template v-slot:body="props">
                    <q-tr :props="props">
                        <q-td v-for="col in props.cols" :key="col.name" :props="props" style="min-width: 120px">
                            <div>{{ col.value }}</div>
                        </q-td>
                        <q-td class="text-right" auto-width v-if="tableButtons.length > 0">
                            <q-btn @click="handleClickOption(btn, props.row, index)" v-for="(btn, index) in tableButtons" :key="index" size="sm" color="grey-6" round dense :icon="btn.icon" class="q-ml-md" padding="5px 5px">
                                <q-tooltip transition-show="scale" transition-hide="scale" anchor="top middle" self="bottom middle" :offset="[10, 10]">
                                    {{ btn.label }}
                                </q-tooltip>
                            </q-btn>
                        </q-td>
                    </q-tr>
                </template>
                <template v-slot:no-data="{ icon }">
                    <div class="full-width row flex-center items-center text-primary q-gutter-sm" style="font-size: 18px">
                        <q-icon size="2em" :name="icon" />
                        <span> 無相關資料 </span>
                    </div>
                </template>
            </q-table>
        </div>
    </q-page>
</template>

<script setup lang="ts">
import axios from 'axios';
import { QTableProps } from 'quasar';
import { ref, computed } from 'vue';
import { useQuasar } from 'quasar'
// import { sort } from 'semver';

interface btnType {
    label: string;
    icon: string;
    status: string;
}

const $q = useQuasar();

const sort = key => {
    if (sortBy.value.key === key) {
        if (sortBy.value.order === 'asc') {
            sortBy.value.order = 'desc';
        } else {
            sortBy.value.key = '';
            sortBy.value.order = '';
        }
    } else {
        sortBy.value.key = key;
        sortBy.value.order = 'asc';
    }
};
const sortBy = ref({ key: '', order: '' });
const filterAge = ref(Infinity);

const blockData = ref([
    {
        name: 'test',
        age: 25,
    },
]);
const sortedData = computed(() => {
    if (sortBy.value.key === '' || sortBy.value.order === '') return blockData.value.slice();
    return blockData.value.slice().sort((a, b) => {
        if (sortBy.value.order === 'asc') {
            return a[sortBy.value.key] > b[sortBy.value.key] ? 1 : -1;
        } else {
            return a[sortBy.value.key] < b[sortBy.value.key] ? 1 : -1;
        }
    });
});
const filteredData = computed(() => {
    return sortedData.value.filter(e => e.age < filterAge.value);
});

const tableConfig = ref([
    {
        label: '姓名',
        name: 'name',
        field: 'name',
        align: 'left',
    },
    {
        label: '年齡',
        name: 'age',
        field: 'age',
        align: 'left',
    },
]);
const tableButtons = ref([
    {
        label: '編輯',
        icon: 'edit',
        status: 'edit',
    },
    {
        label: '刪除',
        icon: 'delete',
        status: 'delete',
    },
]);

const comfirmBtnText = computed(() => {
    return editRow.value < 0 ? '新增' : '更新';
});
const editRow = ref(-1); // 0: create, 1: update
const tempData = ref({
    name: '',
    age: '',
});
async function handleClickOption(btn, data) {
    console.log(btn, data);
    const action = btn.status;
    const dataIndex = blockData.value.indexOf(data);

    switch (action) {
        case 'edit':
            editRow.value = dataIndex;
            tempData.value.name = data.name;
            tempData.value.age = data.age;
            break;
        case 'delete':
            $q.dialog({
                title: '提示',
                message: '是否確定刪除該筆資料？',
                ok: {
                    label: '確定',
                    push: true,
                    color: 'negative',
                },
                cancel: {
                    label: '取消',
                    push: true,
                    color: 'grey-6',
                },
            }).onOk(() => {
                blockData.value.splice(dataIndex, 1);
            });

            break;
        default:
            break;
    }
}

function createData() {
    // check if name is empty
    if (!tempData.value.name) {
        $q.notify({
            type: 'negative',
            message: '請輸入姓名',
            timeout: 1000,
        });
        return;
    }

    // check if not number or age < 0 or age is not integer
    if (!tempData.value.age || isNaN(Number(tempData.value.age)) || Number(tempData.value.age) < 0 || !Number.isInteger(Number(tempData.value.age))) {
        $q.notify({
            type: 'negative',
            message: '請輸入正確年齡',
            timeout: 1000,
        });
        return;
    }

    if (editRow.value >= 0) {
        blockData.value[editRow.value] = {
            name: tempData.value.name,
            age: tempData.value.age,
        };
        return;
    } else {
        blockData.value.push({
            name: tempData.value.name,
            age: tempData.value.age,
        });
    }
}

function cancelEditData() {
    editRow.value = -1;
    tempData.value.name = '';
    tempData.value.age = '';
}

function getData() {
    // get
    axios.get('https://demo.mercuryfire.com.tw:49110/crudTest/a').then((res) => {
        $q.dialog({
            title: '取得資料成功',
            message: JSON.stringify(res.data),
        });
    });
}

function postData() {
    axios.post('https://demo.mercuryfire.com.tw:49110/crudTest', {
        name: 'test',
        age: 25,
    }).then((res) => {
        $q.dialog({
            title: '新增資料成功',
            message: JSON.stringify(res.data),
        });
    });
}

function patchData() {
    axios.patch('https://demo.mercuryfire.com.tw:49110/crudTest', {
        id: 1,
    }).then((res) => {
        $q.dialog({
            title: '更新資料成功',
            message: JSON.stringify(res.data),
        });
    });
}

function deleteData() {
    axios.delete('https://demo.mercuryfire.com.tw:49110/crudTest/1').then((res) => {
        $q.dialog({
            title: '刪除資料成功',
            message: JSON.stringify(res.data),
        });
    });
}

</script>

<style lang="scss" scoped>
.q-table th {
    font-size: 20px;
    font-weight: bold;
}

.q-table tbody td {
    font-size: 18px;
}
</style>
