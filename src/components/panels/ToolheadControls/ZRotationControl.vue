<template>
    <div>
        <div class="v-subheader text--secondary px-0">
            <v-icon small class="mr-2">{{ mdiRotateRight }}</v-icon>
            <span>{{ $t('Panels.ZRotationPanel.Headline') }}</span>
        </div>
        <v-row dense>
            <v-col cols="6">
                <v-text-field
                    v-model.number="amount"
                    :label="$t(mode.amountLabel)"
                    :suffix="mode.unit"
                    type="number"
                    min="0.001"
                    :step="mode.amountStep"
                    :rules="rotationNumberRules"
                    outlined
                    dense
                    hide-details="auto" />
            </v-col>
            <v-col cols="6">
                <v-text-field
                    v-model.number="speed"
                    :label="$t('Panels.ZRotationPanel.Speed')"
                    :suffix="mode.unit ? `${mode.unit}/s` : '/s'"
                    type="number"
                    min="0.001"
                    :step="mode.speedStep"
                    :rules="rotationNumberRules"
                    outlined
                    dense
                    hide-details="auto" />
            </v-col>
        </v-row>
        <div
            class="d-flex flex-wrap my-2"
            role="group"
            :aria-label="$t('Panels.ZRotationPanel.Presets', { amount: $t(mode.amountLabel) })">
            <v-btn
                v-for="preset in mode.presets"
                :key="preset"
                small
                depressed
                class="mr-1 mb-1 px-2"
                :color="Number(amount) === preset ? 'primary' : undefined"
                :aria-pressed="Number(amount) === preset"
                @click="amount = preset">
                {{ preset }}{{ mode.unit }}
            </v-btn>
        </div>
        <v-row dense>
            <v-col cols="6">
                <v-btn block :disabled="rotationDisabled" :loading="rotating" @click="rotate(-1)">
                    <v-icon left small>{{ mdiRotateLeft }}</v-icon>
                    {{ $t(mode.backward) }}
                </v-btn>
            </v-col>
            <v-col cols="6">
                <v-btn block :disabled="rotationDisabled" :loading="rotating" @click="rotate(1)">
                    <v-icon left small>{{ mdiRotateRight }}</v-icon>
                    {{ $t(mode.forward) }}
                </v-btn>
            </v-col>
        </v-row>
        <div class="text-caption text--secondary mt-2">{{ $t(mode.hint) }}</div>
    </div>
</template>

<script lang="ts">
import { Component, Mixins, Watch } from 'vue-property-decorator'
import BaseMixin from '@/components/mixins/base'
import { mdiRotateLeft, mdiRotateRight } from '@mdi/js'

interface RotationMode {
    unit: string
    amountLabel: string
    amount: number
    amountStep: number
    speed: number
    speedStep: number
    presets: number[]
    backward: string
    forward: string
    hint: string
}

// One motor, two meanings. With the vacuum head Klipper drives it in degrees and it turns the
// nozzle; with the dispenser it is the auger and the same ROTATE moves microlitres. The units
// and the defaults follow the head: 300 is a rotation speed, not an auger speed. With no head
// detected the motor keeps the last head's settings, so the cautious numbers are offered.
const NOZZLE_MODE: RotationMode = {
    unit: '°',
    amountLabel: 'Panels.ZRotationPanel.Angle',
    amount: 15,
    amountStep: 1,
    speed: 300,
    speedStep: 10,
    presets: [1, 5, 15, 45, 90, 180, 360],
    backward: 'Panels.ZRotationPanel.Counterclockwise',
    forward: 'Panels.ZRotationPanel.Clockwise',
    hint: 'Panels.ZRotationPanel.NozzleHint',
}
const AUGER_MODE: RotationMode = {
    unit: 'µl',
    amountLabel: 'Panels.ZRotationPanel.Volume',
    amount: 1,
    amountStep: 0.1,
    speed: 0.3,
    speedStep: 0.1,
    presets: [0.1, 0.5, 1, 2, 5, 10],
    backward: 'Panels.ZRotationPanel.Retract',
    forward: 'Panels.ZRotationPanel.Advance',
    hint: 'Panels.ZRotationPanel.AugerHint',
}
const NO_HEAD_MODE: RotationMode = {
    ...AUGER_MODE,
    unit: '',
    amountLabel: 'Panels.ZRotationPanel.Amount',
    backward: 'Panels.ZRotationPanel.Negative',
    forward: 'Panels.ZRotationPanel.Positive',
    hint: 'Panels.ZRotationPanel.NoHeadHint',
}
const ROTATION_LOADING = 'zRotation'
// ROTATE serializes its amount to three decimal places.
const MIN_ROTATION_INPUT = 0.001

@Component
export default class ZRotationControl extends Mixins(BaseMixin) {
    mdiRotateLeft = mdiRotateLeft
    mdiRotateRight = mdiRotateRight
    amount: number | string = NOZZLE_MODE.amount
    speed: number | string = NOZZLE_MODE.speed
    rotationNumberRules = [
        (value: number | string) =>
            (Number.isFinite(Number(value)) && Number(value) >= MIN_ROTATION_INPUT) ||
            this.$t('Panels.ZRotationPanel.MinimumAmount', { minimum: MIN_ROTATION_INPUT }),
    ]

    get mode(): RotationMode {
        const head = this.$store.state.printer['binary_head pnp_motor']
        if (head?.connected_head_id === 0 && head?.connected_head === 'pnp_motor') return NOZZLE_MODE
        return head?.connected_head ? AUGER_MODE : NO_HEAD_MODE
    }

    get rotating(): boolean {
        return this.loadings.includes(ROTATION_LOADING)
    }

    get rotationDisabled(): boolean {
        return (
            !this.klipperReadyForGui ||
            this.printerIsPrinting ||
            this.loadings.length > 0 ||
            !Number.isFinite(Number(this.amount)) ||
            Number(this.amount) < MIN_ROTATION_INPUT ||
            !Number.isFinite(Number(this.speed)) ||
            Number(this.speed) < MIN_ROTATION_INPUT
        )
    }

    // A value typed for one head means something else for the other.
    @Watch('mode', { immediate: true })
    onModeChanged(mode: RotationMode): void {
        this.amount = mode.amount
        this.speed = mode.speed
    }

    rotate(direction: -1 | 1): void {
        if (this.rotationDisabled) return

        // ROTATE resets its own origin. Its F parameter is the speed in the head's units per second.
        const gcode = `ROTATE ANGLE=${direction * Number(this.amount)} F=${Number(this.speed)}`
        this.$store.dispatch('server/addEvent', { message: gcode, type: 'command' })
        this.$socket.emit('printer.gcode.script', { script: gcode }, { loading: ROTATION_LOADING })
    }
}
</script>
