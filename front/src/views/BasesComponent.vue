<template>
    <LayoutComponent title="Bases">
        <template #content>
            <PanelInputFileTextButtonComponent @submit="submit" @upload="fileUpload" v-model="baseTextInput" what="bases"/>
            <p class="text-danger">{{ errorSub }}</p>
            <div style="width:100%; overflow-y: scroll; margin-bottom: 10px">
            <div>
                <p class="text-center">Bases List</p>
                <v-data-table-virtual
                    class="table"
                    v-model:items-per-page="tableClasses"
                    :headers="basesColumns"
                    :items="tableDatas"
                    :class="tableClasses"
                    :items-length="tableDatas.length"
                    :loading="isLoadingSessions"
                    item-value="name"
                />                
                <!-- handleClick -->
                <ButtonComponent @click-handler="exportBasesToTxt" class-names="mt-3" button-text="Export Bases to TXT" />
            </div>

            <div class="container-fluid dummy-form">
                <div class="row tables-container">
                    <div class="col-lg-5 bases-list">
                        <h4 class="text-center">Bases List</h4>
                        <DataTable :data="basesData" :columns="basesColumns" :class="tableClasses" @click="handleClick"></DataTable>
                        <ButtonComponent @click-handler="exportBasesToTxt" class-names="mt-3" button-text="Export Bases to TXT" />
                    </div>
                    <div class="col-lg-7 base-view">
                        <h4 class="text-center">Base View</h4>
                        <DataTable :data="materials" :columns="columns" :class="tableClasses"></DataTable>
                        <ButtonComponent @click-handler="exportMaterialsToTxt" class-names="mt-3" button-text="Export Materials to TXT" />
                    </div>
                </div>
            </div>
        </div>
        <ModalViewComponent ref="modal"></ModalViewComponent>
        </template>
    </LayoutComponent>
</template>

<script>
import axios from 'axios';
import ModalViewComponent from '@/components/components/ModalViewComponent.vue';
import ButtonComponent from '@/ui/ButtonComponent.vue';
import DataTable from 'datatables.net-vue3';
import LayoutComponent from '@/components/LayoutComponent.vue';
import PanelInputFileTextButtonComponent from '@/components/PanelInputFileTextButtonComponent.vue';

export default {
    components: {
        ModalViewComponent,
        DataTable,
        ButtonComponent,
        LayoutComponent,
        PanelInputFileTextButtonComponent
    },
    data() {
        return {
            name: '',
            errorSub: '',
            file: null,
            baseTextInput: '',
            materials: [],
            columns: [
                { title: 'ID', data: 'id' },
                { title: 'First Name', data: 'firstname' },
                { title: 'Last Name', data: 'lastname' },
                { title: 'Email', data: 'email' },
                { title: 'DELETE', data: 'id', render: (data) => {
                    return `<button class="btn btn-danger" data-action="delete" data-id="${data}"><i class="bi bi-dash"></i></button>`;
                }}
            ],
            basesData: [],
            basesColumns: [
                { 
                    title: 'Select',
                    data: 'id',
                    render: (data, type, row) => {
                        return `<input type="checkbox" class="form-check-input" data-id="${data}" data-url="${row.basename}" ${this.selectedBases.some(item => item.id === data) ? 'checked' : ''}>`;
                    },
                    className: 'data-cell'
                },
                { title: 'ID', data: 'id' ,className: 'data-cell'},
                { title: 'Base Name', data: 'basename' ,className: 'data-cell'},
                { title: 'Status', data: 'status',className: 'data-cell', render: (data) => {
                    return `<a href="#" class="btn btn-${data}">${data}</a>`;
                }},
                { title: 'View', data: 'id', render: (data) => {
                    return `<button class="btn btn-primary" data-action="view" data-id="${data}"><i class="bi bi-plus"></i></button>`;
                }},
                { title: 'DELETE', data: 'id', render: (data) => {
                    return `<button class="btn btn-danger" data-action="deletebasebylist" data-id="${data}"><i class="bi bi-dash"></i></button>`;
                }}
            ],
            tableClasses: 'table text-start align-middle table-bordered table-hover mb-0',
            selectedBases: []
        };
    },
    methods: {
        getMaterials() {
            const currentSessionName = this.getCurrentSessionName();
            if (!currentSessionName) {
                this.errorSubTemplates = 'No session loaded';
                return;
            }
            axios.get(`${import.meta.env.VITE_BACK_URL}/api/get/list/bases/${currentSessionName}`)
                .then(res => {
                    this.basesData = res.data.map(item => ({
                        id: item.id,
                        basename: item.name,
                        status: item.status,
                    }));
                })
                .catch(error => {
                    console.error('Error fetching materials:', error);
                });
        },
        getCurrentSessionName() {
            const name = document.cookie.split('; ').find(row => row.startsWith('currentSessionName='));
            return name ? name.split('=')[1] : null;
        },
        handleClick(event) {
            const action = event.target.getAttribute('data-action');
            const id = event.target.getAttribute('data-id');

            if (action === 'view') {
                this.view(id);
            } else if (action === 'delete') {
                this.del(id);
            }
        },
        submit() {
            const currentSessionName = this.getCurrentSessionName();
            if (!currentSessionName) {
                this.errorSubTemplates = 'No session loaded';
                return;
            }
            let token = '';
            let cookies = document.cookie.split(";");
            cookies.forEach(cookie => {
                if (cookie.includes('authToken')) {
                    token = cookie.split("=")[1].trim();
                }
            });
            this.errorSub = '';
            if (this.file || this.baseTextInput) {
                let baseArray = this.baseTextInput.split('\n').map(base => base.trim()).filter(base => base);
                let fileContent = this.file ? this.file : baseArray.join('\n');
                axios.post(`${import.meta.env.VITE_BACK_URL}/api/input/material`, {
                    token: token,
                    session: currentSessionName,
                    type: 'bases',
                    file: fileContent
                }).then(res => {
                    this.getMaterials();
                    this.$refs.inputEl1.value = '';
                    this.baseTextInput = '';
                    if (res.data.data === 'error') {
                        this.errorSub = res.data.error;
                    }
                }).catch(error => {
                    this.errorSub = 'An error occurred during submission.';
                    console.error('Submission error:', error);
                });
            } else {
                this.errorSub = 'Fill in the fields!';
            }
        },
        del(id) {
            let token = '';
            let cookies = document.cookie.split(";");
            cookies.forEach(cookie => {
                if (cookie.includes('authToken')) {
                    token = cookie.split("=")[1].trim();
                }
            });
            axios.post(`${import.meta.env.VITE_BACK_URL}/api/del/material`, {
                token: token,
                id: id,
                type: 'bases'
            }).then(res => {
                this.getMaterials();
            }).catch(error => {
                console.error('Error deleting material:', error);
            });
        },
        get_zip_data(zip, files_keys, data, callback) {
            zip.files[files_keys[0]].async("string").then(el_data => {
                files_keys.shift();
                data += '\n' + el_data;
                if (files_keys.length > 0) {
                    this.get_zip_data(zip, files_keys, data, callback);
                } else {
                    this.file = data;
                    callback(data);
                }
            });
        },
        fileUpload(event) {
            const fileId = event.target.getAttribute('id');
            const file = event.target.files[0];
            if (fileId === 'formFile1') {
                file.arrayBuffer().then(buffer => {
                    const bufferByteLength = buffer.byteLength;
                    const bufferUint8Array = new Uint8Array(buffer, 0, bufferByteLength);
                    this.file = new TextDecoder().decode(bufferUint8Array);
                }).catch(error => {
                    console.error('Error reading file:', error);
                });
            } else if (fileId === 'formFile2') {
                this.$refs.inputEl1.value = '';
                file.arrayBuffer().then(buffer => {
                    const zip = new JSZip();
                    zip.loadAsync(buffer).then(zip => {
                        let data = '';
                        const files = zip.files;
                        const files_keys = Object.keys(files);
                        this.get_zip_data(zip, files_keys, data, () => {});
                    }).catch(error => {
                        console.error('Error reading zip file:', error);
                    });
                }).catch(error => {
                    console.error('Error reading file:', error);
                });
            }
        },
        view(id) {
            const type = 'bases';
            axios.get(`${import.meta.env.VITE_BACK_URL}/api/get/materials/${id}/${type}/*`)
                .then(res => {
                    this.materials = res.data.map(item => ({
                        id: item[0],
                        firstname: item[1],
                        lastname: item[2],
                        email: item[3],
                    }));
                }).catch(error => {
                    console.error('Error fetching material details:', error);
                });
        },
        exportBasesToTxt() {
            const headers = this.basesColumns.map(col => col.title).join('\t');
            const rows = this.basesData.map(base => Object.values(base).join('\t')).join('\n');
            const txtContent = `${headers}\n${rows}`;
            const blob = new Blob([txtContent], { type: 'text/plain' });
            const link = document.createElement('a');
            link.href = URL.createObjectURL(blob);
            link.download = 'bases.txt';
            link.click();
        },
        exportMaterialsToTxt() {
            const headers = this.columns.map(col => col.title).join('\t');
            const rows = this.materials.map(material => Object.values(material).join('\t')).join('\n');
            const txtContent = `${headers}\n${rows}`;
            const blob = new Blob([txtContent], { type: 'text/plain' });
            const link = document.createElement('a');
            link.href = URL.createObjectURL(blob);
            link.download = 'materials.txt';
            link.click();
        },
        toggleSelection(id, url) {
            const index = this.selectedBases.findIndex(item => item.id === id);
            if (index !== -1) {
                this.selectedBases.splice(index, 1);
            } else {
                this.selectedBases.push({ id, url });
            }
            console.log(this.selectedBases);
        },
        saveSelection() {
            localStorage.setItem('selectedBases', JSON.stringify(this.selectedBases));
        },
        loadSelection() {
            this.selectedBases = JSON.parse(localStorage.getItem('selectedBases') || '[]');
        }
    },
    mounted() {
        this.loadSelection();
        this.getMaterials();
        console.log(this.selectedBases);
        document.querySelector('.table').addEventListener('change', (event) => {
            if (event.target.classList.contains('form-check-input')) {
                const id = parseInt(event.target.getAttribute('data-id'));
                const url = event.target.dataset.url;
                this.toggleSelection(id, url);
                this.saveSelection();
            }
        });
    }
}
</script>

<style scoped>
#main-part {
  display: flex;
  flex-direction: column;
  width: 100%;
}
.dummy-form {
    overflow: auto;
    margin-top: 1em;
    width: auto;
}
.data-cell {
    color: #6C7293 !important;
}

.table th {
    color: #fff !important;
}

.table-dark th, .table-dark td {
    border-color: #333;
    color: #ddd;
}

.headerzn {
    color: #ddd;
}
.form-check-input {
    -webkit-appearance: none;
    -moz-appearance: none;
    appearance: none;
    width: 20px;
    height: 20px;
    border-radius: 4px;
    outline: none;
    cursor: pointer;
    background-color: var(--da);
}

.form-check-input:checked {
    background-color: var(--primary);
    border-color: var(--primary);
}

.form-check-input:checked::after {
    display: block;
    color: white;
    font-size: 14px;
    text-align: center;
    line-height: 20px;
}
.btn-dead {
    background-color: var(--primary) !important;
    color: #000;
    width: 100px;
}

.btn-junk {
    background-color: var(--primary) !important;
    color: #000;
    width: 100px;
}
.btn-inbox {
    background-color: #2bff00!important;
    color: #000;
    width: 100px;
}
.btn-none {
    background-color: var(--primary) !important;
    color: #000;
    width: 100px;
}
.btn-checked {
    background-color: #2bff00 !important;
    color: #000;
    width: 100px;
}

.dt-paging-button {
  border: 1px solid white;
  background-color: black;
  margin: 1%;
  border-radius: 7px;
  transition: background-color 0.2s linear;
}

.dt-paging-button:hover {
  background-color: var(--primary);
  color: white;
}

label {
  padding: 1em 1em 1em 0;
}

.dt-input {
  margin-right: 1em;
}

.dt-info {
  padding: 1em 1em 1em 0;
  margin-bottom: 1%;
}

label {
  color: white;
}

.btn-primary {
  background-color: var(--primary) !important;
  border: none;
  transition: background-color 0.2s linear;
}

.btn:hover {
  background-color: #f84545;
}

.tables-container {
    display: flex;
    flex-direction: row;
    gap: 2em;
}

.bases-list, .base-view {
    flex: 1;
}

/deep/ .dt-paging-button {
  border: 1px solid white;
  background-color: transparent;
  margin: 1%;
  border-radius: 7px;
  transition: background-color 0.2s linear;
}

/deep/ .dt-paging-button:hover {
  background-color: red;
  color: white;
}

/deep/ .dt-input {
  margin-right: 1em;
}

/deep/ .dt-info {
  padding: 1em 1em 1em 0;
  margin-bottom: 1%;
}

/deep/ label {
  padding: 1em 1em 1em 0;
}
</style>
