<script>
    const projectEndpoint = '/api/project'
    const personEndpoint = '/api/person'

    export default {
        data() {
            return {
                isValid: false,
                input: {},
                rules: {
                    startsWithLetter: value => {
                        const pattern = /^\p{L}/u
                        return pattern.test(value) || 'Wymagane zaczynanie się od litery'
                    },
                    validDate: value => {
                        const date = new Date(value)
                        return !!date || `Wymagana prawidłowa data`
                    }
                },
                persons: [],
                showTasksDialog: false,
                tasks: []
            }
        },
        props: [ 'project' ],
        emits: [ 'close', 'listChanged' ],
        methods: {
           send() {
              this.input.tasks = this.tasks
              fetch(projectEndpoint, {
                method: 'POST',
                headers: { 'Content-type': 'application/json' },
                body: JSON.stringify({...this.input, tasks: this.tasks})
              }).then(res => {
                res.json().then(data => {
                    if(!res.ok) {
                        this.$emit('close', data.error, 'error')
                    } else {
                        this.input = {}
                        this.tasks = []
                        this.$emit('close', `Projekt ${data.name} - dodano`)
                        this.$emit('listChanged')
                    }
                }).catch(err => {
                    this.$emit('close', 'Dane odrzucone', 'error')
                })
              })
           },
           update() {
              this.input.tasks = this.tasks
              fetch(projectEndpoint, {
                method: 'PUT',
                headers: { 'Content-type': 'application/json' },
                body: JSON.stringify({...this.input, tasks: this.tasks})
              }).then(res => {
                res.json().then(data => {
                    if(!res.ok) {
                        this.$emit('close', data.error, 'error')
                    } else {
                        this.input = {}
                        this.tasks = []
                        this.$emit('close', `Projekt ${data.name} - zaktualizowano`)
                        this.$emit('listChanged')
                    }
                }).catch(err => {
                    this.$emit('close', 'Dane odrzucone', 'error')
                })
              })
           },
           remove() {
              fetch(projectEndpoint + '?' + new URLSearchParams({ _id: this.input._id }), {
                method: 'DELETE'
              }).then(res => {
                res.json().then(data => {
                    if(!res.ok) {
                        this.$emit('close', data.error, 'error')
                    } else {
                        this.input = {}
                        this.tasks = []
                        this.$emit('close', `Projekt ${data.name} - usunięto`)
                        this.$emit('listChanged')
                    }
                }).catch(err => {
                    this.$emit('close', 'Dane odrzucone', 'error')
                })
              })
           },
           setData(data) {
              this.input = {}
              this.tasks = data.tasks || []
              Object.assign(this.input, data)
           },
           clear() {
                this.input = { _id: this.input._id }
                this.tasks = []
                this.isValid = false
           },
           close() {
                this.$emit('close')
           },
           openTasksDialog() {
               this.tasks = [...this.input.tasks || []]
               this.showTasksDialog = true
           },
           saveTasks() {
               this.input.tasks = this.tasks
               this.showTasksDialog = false
           },
           addTask() {
               this.tasks.push({
                   name: '',
                   startDate: '',
                   endDate: '',
                   assignees: []
               })
           },
           removeTask(index) {
               this.tasks.splice(index, 1)
           },
           validateTaskDates(task) {
               if (task.startDate && task.endDate) {
                   return new Date(task.startDate) <= new Date(task.endDate);
               }
               return true;
           }
        },
        mounted() {
            Object.assign(this.input, this.project)
            if (this.project?.tasks) {
                this.tasks = [...this.project.tasks]
            }
            
            fetch(personEndpoint + '?' + 
                    new URLSearchParams({ sort: 'lastName', order: 'asc' }).toString())
                .then(res => res.json().then(facet => {
                    if(!facet.error) {
                        this.persons = facet.data
                    }
                }))
        }
    }
</script>

<template>
    <v-form v-model="isValid">
        <v-card>
            <v-card-title>{{ input._id ? 'Edytuj dane' : 'Wprowadź dane nowej osoby' }}</v-card-title>
            <v-card-subtitle>
                Dane muszą spełniać odpowiednie reguły, zarówno w tym formularzu, jak i w backendzie.
            </v-card-subtitle>
            <v-card-text>
                <v-text-field variant="outlined" label="Nazwa" v-model="input.name" :rules="[ rules.startsWithLetter ]">
                </v-text-field>
                <v-text-field type="date" variant="outlined" label="Data startu" v-model="input.startDate" :rules="[ rules.validDate ]">
                </v-text-field>
                <v-text-field type="date" variant="outlined" label="Data końca" v-model="input.endDate" :rules="[ rules.validDate ]">
                </v-text-field>
                <v-autocomplete variant="outlined"
                    v-model="input.contractor_ids"
                    :items="persons"
                    :item-title="item => item.firstName + ' ' + item.lastName"
                    item-value="_id"
                    label="Wykonawcy"
                    multiple chips
                ></v-autocomplete>
            </v-card-text>
            <v-card-actions>
                <v-spacer></v-spacer>
                <v-btn variant="elevated" @click="openTasksDialog">Zadania</v-btn>
                <v-btn variant="elevated" @click="clear">Zeruj</v-btn>
                <v-btn color="primary" variant="elevated" @click="send" :disabled="!isValid" v-if="!input._id">Wyślij</v-btn>
                <v-btn color="secondary" variant="elevated" @click="update" :disabled="!isValid" v-if="input._id">Aktualizuj</v-btn>
                <v-btn color="error" variant="elevated" @click="remove" v-if="input._id">Usuń</v-btn>
                <v-btn variant="elevated" @click="close">Zamknij</v-btn>
            </v-card-actions>
        </v-card>
    </v-form>

    <v-dialog v-model="showTasksDialog" max-width="800">
        <v-card>
            <v-card-title>Zadania projektu</v-card-title>
            <v-card-text>
                <div v-for="(task, index) in tasks" :key="index" class="task-item mb-4 pa-4">
                    <v-text-field v-model="task.name" label="Nazwa zadania" outlined class="mb-2"></v-text-field>
                    <v-text-field type="date" v-model="task.startDate" label="Data startu" outlined
                        :error-messages="!validateTaskDates(task) ? 'Data startu nie może być późniejsza niż data końca' : ''">
                    </v-text-field>
                    <v-text-field type="date" v-model="task.endDate" label="Data końca" outlined
                        :error-messages="!validateTaskDates(task) ? 'Data końca nie może być wcześniejsza niż data startu' : ''">
                    </v-text-field>
                    
                    <v-autocomplete
                        v-model="task.assignees"
                        :items="persons"
                        :item-title="item => item.firstName + ' ' + item.lastName"
                        item-value="_id"
                        label="Przypisani wykonawcy"
                        multiple
                        chips
                        outlined
                    ></v-autocomplete>

                    <v-alert v-if="task.endDate" type="success" class="mt-2">
                        Zakończone
                    </v-alert>
                    
                    <v-btn icon @click="removeTask(index)" color="error" class="mt-2">
                        <v-icon>mdi-delete</v-icon>
                    </v-btn>
                </div>

                <v-btn @click="addTask" color="primary" class="mt-4">
                    <v-icon left>mdi-plus</v-icon>
                    Dodaj zadanie
                </v-btn>
            </v-card-text>
            <v-card-actions>
                <v-spacer></v-spacer>
                <v-btn @click="showTasksDialog = false">Anuluj</v-btn>
                <v-btn color="success" @click="saveTasks">Zapisz zmiany</v-btn>
            </v-card-actions>
        </v-card>
    </v-dialog>

</template>

<style scoped>
.task-item {
    border: 1px solid #e0e0e0;
    border-radius: 8px;
    background-color: #f5f5f5;
}
</style>