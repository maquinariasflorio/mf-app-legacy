<template>
    <v-container class="mf-page mf-page-report-operator">

        <v-card :loading="loading">
            <v-card-title>{{ title }}</v-card-title>

            <v-card-text>

                <v-form ref="form">

                    <v-select v-model="formData.user"
                              :items="operators"
                              label="Operador"
                              item-value="_id"
                              :disabled="loading"
                              :rules="[ v => !!v || 'El operador es requerido' ]"
                    >

                        <template #item="{ item }">
                            {{ item.rut }} | {{ item.name }}
                        </template>

                        <template #selection="{ item }">
                            {{ item.rut }} | {{ item.name }}
                        </template>

                    </v-select>

                    <mf-date-picker v-model="formData.startDate"
                                    label="Fecha Inicio"
                                    :attrs="{
                                        max: now,
                                    }"
                                    :disabled="loading"
                                    :rules="[ v => !!v || 'La fecha es requerida' ]"
                    />

                    <mf-date-picker v-model="formData.endDate"
                                    label="Fecha Término"
                                    :attrs="{
                                        max: now,
                                    }"
                                    :disabled="loading"
                                    :rules="[ v => !!v || 'La fecha es requerida' ]"
                    />

                    <v-btn block color="primary" :disabled="loading" @click="submit">
                        Descargar
                    </v-btn>

                </v-form>

            </v-card-text>
        </v-card>

    </v-container>
</template>

<script>
import gql from 'graphql-tag'
import moment from 'moment'
import { mapGetters } from 'vuex'
import { Error } from './../static/errors'
import { newEmptyWorkbook, addWorksheet, setExcelHeader, addExcelRow, saveExcelFile, mainColor } from './../static/utils/excel'

moment.locale('es')

export default {
    apollo: {
        operators: {
            query: gql`query {
                getAllUsersWithRoleName(role: "operator") {
                    _id,
                    rut,
                    name,
                    email,
                }
            }`,
            update: (data) => data.getAllUsersWithRoleName,
        },
    },

    data() {

        return {
            loading  : false,
            formData : {
                user      : null,
                startDate : moment().format('YYYY-MM-DD'),
                endDate   : moment().format('YYYY-MM-DD'),
            },
        }

    },

    computed: {
        ...mapGetters( {
            title: 'navbar/getTitle',
        } ),

        now() {

            return moment().toISOString()

        },
    },

    methods: {
        submit() {

            if (this.$refs.form.validate() ) {

                this.loading = true

                this.$apollo.query( {
                    query: gql`query getAllMachineryJobRegistryByUserAndDate($user: String!, $startDate: String!, $endDate: String!) {
                        getAllMachineryJobRegistryByUserAndDate(user: $user, startDate: $startDate, endDate: $endDate) {
                            _id,
                            startHourmeter,
                            endHourmeter,
                            date,
                            totalHours,
                            machineryType,
                            executor {
                                rut,
                                name,
                            }
                        }
                    }`,

                    variables: {
                        ...this.formData,
                    },

                    fetchPolicy: 'network-only',
                } )
                    .then( ( { data: { getAllMachineryJobRegistryByUserAndDate } } ) => {

                        this.generateExcelFile(getAllMachineryJobRegistryByUserAndDate)
                        this.loading = false

                    } )
                    .catch( (error) => {

                        console.warn(error)
                        this.$alert(Error.DEFAULT_ERROR_MESSAGE, 'error')
                        this.loading = false

                    } )

            }

        },

        generateExcelFile(data) {

            const workbook = newEmptyWorkbook()

            // 1: sort by date
            const sortedData = data.sort( (a, b) => new Date(a.date) - new Date(b.date) )

            // 2: filter by OTHER
            const machineryData = sortedData.filter( (item) => item.machineryType === 'OTHER')

            // 3: add worksheet

            if (machineryData.length > 0) {

                const worksheet = addWorksheet(workbook, 'Operador')
                setExcelHeader(workbook, worksheet)

                addExcelRow(workbook, worksheet, [ `Operador: ${machineryData[0].executor.rut} | ${machineryData[0].executor.name}` ], { isHeader: true, bordered: false } )

                const headers = [ 'Fecha', 'Horómetro Inicial', 'Horómetro Final', 'Horas Máquina', 'Total Horas', 'Semanas Corridas' ]
                addExcelRow(workbook, worksheet, headers, { isHeader: true } )

                let currentWeek = 0
                let sumHours = 0
                let sumDays = 0

                let totalHours = 0
                let totalDays = 0

                for (const [ index, item ] of machineryData.entries() ) {

                    currentWeek = moment.utc(item.date).week()

                    const isSaturday = moment.utc(item.date).day() === 6
                    const isLastDayOfWeek = machineryData[index + 1] && moment.utc(machineryData[index + 1].date).week() !== currentWeek ? true : false

                    sumDays++
                    totalDays++
                    sumHours += isSaturday ? item.totalHours * 2 : item.totalHours
                    totalHours += isSaturday ? item.totalHours * 2 : item.totalHours

                    const newRow = [
                        moment.utc(item.date).format('dddd DD [de] MMMM [de] YYYY'),
                        item.startHourmeter,
                        item.endHourmeter,
                        item.totalHours,
                        isSaturday ? item.totalHours * 2 : item.totalHours,
                        isLastDayOfWeek || index === (machineryData.length - 1) ? (Math.round( (sumHours / sumDays) * 10) / 10) : 0,
                    ]

                    const { row } = addExcelRow(workbook, worksheet, newRow)

                    row.eachCell( (cell, colNumber) => {

                        cell.alignment = { horizontal: 'center' }

                    } )

                    sumHours = isLastDayOfWeek ? 0 : sumHours
                    sumDays = isLastDayOfWeek ? 0 : sumDays

                }

                // add total
                let firstCol = 0
                const { row } = addExcelRow(workbook, worksheet, [ '', '', '', 'Total horas: ', totalHours, (Math.round( (totalHours / totalDays) * 10) / 10) ], { bordered: false } )
                row.eachCell( (cell, colNumber) => {

                    if (firstCol === 0 && cell.value)
                        firstCol = colNumber

                    cell.border = {
                        top    : firstCol > 0 && colNumber >= firstCol ? { style: 'thin', color: { argb: mainColor } } : undefined,
                        left   : colNumber === firstCol ? { style: 'thin', color: { argb: mainColor } } : undefined,
                        bottom : firstCol > 0 && colNumber >= firstCol ? { style: 'thin', color: { argb: mainColor } } : undefined,
                        right  : colNumber === row.cellCount ? { style: 'thin', color: { argb: mainColor } } : undefined,
                    }

                    cell.font = {
                        bold  : false,
                        color : { argb: mainColor },
                    }

                    cell.alignment = { horizontal: 'center' }

                } )

                saveExcelFile(workbook, 'horas_trabajadas')

            }
            else {

                this.$alert('No hay datos para generar el archivo', 'info')

            }

        },
    },
}
</script>