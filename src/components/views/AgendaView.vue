<template>
  <div class="agenda-view">
    <div class="agenda-header">
      <h3 class="text-h6">
        {{ currentDate.format('dddd, MMMM D, YYYY') }}
      </h3>
      <v-chip
        v-if="dayEvents.length > 0"
        color="primary"
        size="small"
      >
        {{ dayEvents.length }} event{{ dayEvents.length > 1 ? 's' : '' }}
      </v-chip>
    </div>

    <div class="agenda-content">
      <div
        v-if="dayEvents.length === 0"
        class="no-events"
      >
        <v-icon
          icon="mdi-calendar-blank"
          size="64"
          color="grey-lighten-2"
          class="mb-4"
        />
        <p class="text-body-1 text-grey">
          No events scheduled for this day
        </p>
      </div>

      <div
        v-else
        class="events-list"
      >
        <v-card
          v-for="event in sortedEvents"
          :key="event.id"
          class="agenda-event mb-4"
          :style="{
            borderLeft: `4px solid ${getEventColor(event)}`
          }"
          :class="{
            'event-completed': event.status === 'completed',
            'event-cancelled': event.status === 'cancelled',
            'event-overdue': isEventOverdue(event)
          }"
          elevation="2"
          @click="handleEventClick({ event, nativeEvent: $event})"
        >
          <v-card-text class="pa-4">
            <div class="event-header d-flex align-center mb-2">
              <v-icon
                v-if="event.icon"
                :icon="event.icon"
                :color="getEventColor(event)"
                class="mr-2"
              />
              <h4 class="event-title text-h6 flex-grow-1">
                {{ event.title }}
              </h4>
              <v-chip
                :color="getStatusColor(event.status || 'open')"
                size="small"
                variant="outlined"
              >
                {{ formatStatus(event.status || 'open') }}
              </v-chip>
            </div>

            <div
              v-if="event.subtitle"
              class="event-subtitle text-subtitle-1 mb-2"
            >
              {{ event.subtitle }}
            </div>

            <div class="event-time d-flex align-center mb-2">
              <v-icon
                icon="mdi-clock-outline"
                size="small"
                class="mr-2"
              />
              <span class="text-body-2">{{ formatEventTime(event) }}</span>
              <span
                v-if="!event.isAllDay"
                class="ml-2 text-caption text-grey"
              >
                ({{ getEventDurationText(event) }})
              </span>
            </div>

            <div
              v-if="event.location"
              class="event-location d-flex align-center mb-2"
            >
              <v-icon
                icon="mdi-map-marker"
                size="small"
                class="mr-2"
              />
              <span class="text-body-2">{{ event.location }}</span>
            </div>

            <div
              v-if="event.description"
              class="event-description"
            >
              <p class="text-body-2 mb-0">
                {{ event.description }}
              </p>
            </div>

            <div
              v-if="event.data && Object.keys(event.data).length > 0"
              class="event-metadata mt-3"
            >
              <v-divider class="mb-2" />
              <div class="d-flex flex-wrap gap-2">
                <v-chip
                  v-for="(value, key) in event.data"
                  :key="key"
                  size="x-small"
                  variant="outlined"
                >
                  {{ key }}: {{ value }}
                </v-chip>
              </div>
            </div>
          </v-card-text>

          <v-card-actions class="pa-4 pt-0">
            <v-spacer />
            <v-btn
              v-if="event.status !== 'completed' && event.status !== 'cancelled'"
              variant="text"
              size="small"
              color="success"
              @click.stop="markAsCompleted(event)"
            >
              Mark Complete
            </v-btn>
            <v-btn
              v-if="event.status !== 'cancelled'"
              variant="text"
              size="small"
              color="error"
              @click.stop="markAsCancelled(event)"
            >
              Cancel
            </v-btn>
          </v-card-actions>
        </v-card>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
  import { computed } from 'vue'
  import dayjs, { type Dayjs } from 'dayjs'
  import { VIcon } from 'vuetify/components/VIcon'
  import type {
    CalendarEventInternal,
    CalendarConfig,
    EventClickHandler,
    EventStatus,
    CalendarEvent,
    EventClickData
  } from '@/types'
  import {
    getEventsForDay,
    getEventColor,
    getEventDuration,
    formatEventTime,
    sortEventsByStartTime
  } from '@/utils'

  // Component registration for library usage
  defineOptions({
    components: {
      VIcon
    }
  })

  export interface AgendaViewProps {
    events: CalendarEventInternal[]
    currentDate: Dayjs
    config: CalendarConfig
  }

  export interface AgendaViewEmits {
    (e: 'event-click', data: EventClickData): void
    (e: 'event-update', eventId: string, updates: Partial<CalendarEvent>): void
  }

  const props = defineProps<AgendaViewProps>()
  const emit = defineEmits<AgendaViewEmits>()

  const dayEvents = computed(() => {
    return getEventsForDay(props.events, props.currentDate)
  })

  const sortedEvents = computed(() => {
    return sortEventsByStartTime(dayEvents.value)
  })

  const getStatusColor = (status: EventStatus): string => {
    switch (status) {
      case 'completed':
        return 'success'
      case 'cancelled':
        return 'error'
      case 'overdue':
        return 'warning'
      case 'planned':
        return 'info'
      case 'open':
      default:
        return 'default'
    }
  }

  const formatStatus = (status: EventStatus): string => {
    return status.charAt(0).toUpperCase() + status.slice(1)
  }

  const getEventDurationText = (event: CalendarEventInternal): string => {
    const duration = getEventDuration(event)
    const hours = Math.floor(duration / 60)
    const minutes = duration % 60
    
    if (hours > 0 && minutes > 0) {
      return `${hours}h ${minutes}m`
    } else if (hours > 0) {
      return `${hours}h`
    } else {
      return `${minutes}m`
    }
  }

  const isEventOverdue = (event: CalendarEventInternal): boolean => {
    return event.endDate.isBefore(dayjs()) && event.status !== 'completed' && event.status !== 'cancelled'
  }

  const handleEventClick: EventClickHandler = (data) => {
    emit('event-click', data)
  }

  const markAsCompleted = (event: CalendarEventInternal) => {
    emit('event-update', event.id, { status: 'completed' })
  }

  const markAsCancelled = (event: CalendarEventInternal) => {
    emit('event-update', event.id, { status: 'cancelled' })
  }
</script>

<style scoped>
  .agenda-view {
    height: 100%;
    display: flex;
    flex-direction: column;
    padding: 16px;
  }

  .agenda-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 24px;
    padding-bottom: 16px;
    border-bottom: 1px solid rgba(0, 0, 0, 0.12);
  }

  .agenda-content {
    flex: 1;
    overflow-y: auto;
  }

  .no-events {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    height: 300px;
    text-align: center;
  }

  .events-list {
    max-width: 800px;
    margin: 0 auto;
  }

  .agenda-event {
    transition: transform 0.2s, box-shadow 0.2s;
    cursor: pointer;
  }

  .agenda-event:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15) !important;
  }

  .agenda-event.event-completed {
    opacity: 0.7;
  }

  .agenda-event.event-completed .event-title {
    text-decoration: line-through;
  }

  .agenda-event.event-cancelled {
    opacity: 0.5;
  }

  .agenda-event.event-cancelled .event-title {
    text-decoration: line-through;
  }

  .agenda-event.event-overdue {
    border-left-color: #f44336 !important;
  }

  .event-header {
    align-items: flex-start;
  }

  .event-title {
    line-height: 1.2;
  }

  .event-subtitle {
    color: rgba(0, 0, 0, 0.7);
    font-style: italic;
  }

  .event-description {
    color: rgba(0, 0, 0, 0.8);
    line-height: 1.4;
  }

  .event-metadata {
    margin-top: 12px;
  }

  .gap-2 {
    gap: 8px;
  }
</style>
