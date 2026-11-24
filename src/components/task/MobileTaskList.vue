<template>
    <!-- 该组件使用 render 函数渲染，故 template 为空 -->
</template>

<script lang="ts">
import { defineComponent, h, type PropType } from 'vue'
import {
    NCheckbox,
    NTag,
    NProgress,
    NEllipsis,
    type TagProps,
} from 'naive-ui'
import type { TaskRecord } from '../../types'
import { renderActions } from './TaskRowActions'
import type { TaskAction, TaskActionExtra } from './TaskRowActions'
import AppIcon from '../common/AppIcon.vue'

export default defineComponent({
    name: 'MobileTaskList',
    props: {
        tasks: {
            type: Array as PropType<TaskRecord[]>,
            required: true,
        },
        selectedRowKeys: {
            type: Array as PropType<string[]>,
            required: true,
        },
        isAndroid: {
            type: Boolean,
            default: false,
        },
    },
    emits: ['update:selectedRowKeys', 'action'],
    setup(props, { emit }) {
        function formatSpeed(bytesPerSec: number): string {
            if (!bytesPerSec || bytesPerSec <= 0) return ''
            const units = ['B/s', 'KB/s', 'MB/s', 'GB/s']
            let unitIndex = 0
            let value = bytesPerSec
            while (value >= 1024 && unitIndex < units.length - 1) {
                value /= 1024
                unitIndex++
            }
            return `${value.toFixed(1)} ${units[unitIndex]}`
        }

        function progressPercent(task: TaskRecord): number {
            if (task.status === 'completed' || task.status === 'processing')
                return 100
            if (task.fileSize > 0) {
                return Math.min(
                    100,
                    Math.round((task.downloaded / task.fileSize) * 100),
                )
            }
            return 0
        }

        const statusMap: Record<
            string,
            { type: TagProps['type']; label: string }
        > = {
            waiting: { type: 'info', label: '等待中' },
            downloading: { type: 'info', label: '下载中' },
            paused: { type: 'warning', label: '暂停' },
            completed: { type: 'success', label: '已完成' },
            error: { type: 'error', label: '错误' },
            processing: { type: 'info', label: '处理中' },
            interrupted: { type: 'warning', label: '已中断' },
        }

        function actionNodes(task: TaskRecord) {
            return renderActions(task, {
                emit: (
                    action: TaskAction,
                    taskId: string,
                    extra?: TaskActionExtra,
                ) => {
                    emit('action', action, taskId, extra)
                },
                isAndroid: props.isAndroid,
            })
        }

        function toggleSelection(taskId: string, checked: boolean) {
            const newKeys = new Set(props.selectedRowKeys)
            if (checked) {
                newKeys.add(taskId)
            } else {
                newKeys.delete(taskId)
            }
            emit('update:selectedRowKeys', Array.from(newKeys))
        }

        function renderBottomLeft(task: TaskRecord) {
            if (task.status === 'error' || task.status === 'interrupted') {
                return h(
                    'div',
                    {
                        class: [
                            'task-card-status-message',
                            task.status === 'error'
                                ? 'is-error'
                                : 'is-interrupted',
                        ],
                    },
                    task.status === 'interrupted'
                        ? '上次运行中断，等待恢复'
                        : task.errorMsg || '下载失败',
                )
            }

            if (task.status === 'downloading' || task.status === 'paused') {
                const percent = progressPercent(task)
                return h('div', { class: 'task-card-progress-inline' }, [
                    h(NProgress, {
                        percentage: percent,
                        showIndicator: false,
                        height: 6,
                        borderRadius: 999,
                    }),
                    h(
                        'span',
                        { class: 'task-card-percent' },
                        task.status === 'downloading' &&
                            task.speed &&
                            task.speed > 0
                            ? `${percent}% · ${formatSpeed(task.speed)}`
                            : `${percent}%`,
                    ),
                ])
            }

            if (props.isAndroid && task.filePath) {
                return h('div', { class: 'task-card-filepath' }, [
                    h(
                        NEllipsis,
                        {
                            style: {
                                fontSize: '11px',
                                maxWidth: '100%',
                            },
                            expandTrigger: 'click',
                            lineClamp: 1,
                            tooltip: false,
                        },
                        () => task.filePath,
                    ),
                ])
            }

            return null
        }

        function renderTaskCard(task: TaskRecord) {
            const status = statusMap[task.status] || {
                type: 'default' as const,
                label: task.status,
            }
            const checked = props.selectedRowKeys.includes(task.id)
            const disabled = task.status === 'downloading'
            const actions = actionNodes(task)
            const leftNode = renderBottomLeft(task)

            const metaParts = [task.artist || '未知歌手']
            if (task.quality) metaParts.push(task.quality)

            const cardChildren = [
                h('div', { class: 'task-card-top' }, [
                    h(NCheckbox, {
                        checked,
                        disabled,
                        'onUpdate:checked': (val: boolean) =>
                            toggleSelection(task.id, val),
                    }),
                    task.coverUrl
                        ? h('img', {
                              src: task.coverUrl,
                              alt: task.songTitle,
                              class: 'task-card-cover',
                              loading: 'lazy',
                          })
                        : h(
                              'div',
                              {
                                  class: 'task-card-cover task-card-cover-fallback',
                              },
                              [h(AppIcon, { name: 'music', size: 15 })],
                          ),
                    h('div', { class: 'task-card-song' }, [
                        h(
                            'div',
                            { class: 'task-card-title' },
                            task.songTitle || '未知歌曲',
                        ),
                        h(
                            'div',
                            { class: 'task-card-sub' },
                            metaParts.join(' · '),
                        ),
                    ]),
                    h(
                        NTag,
                        {
                            type: status.type,
                            size: 'tiny',
                            round: true,
                            bordered: false,
                            class: 'task-status-tag',
                        },
                        () => status.label,
                    ),
                ]),
            ]

            if (leftNode || actions.length > 0) {
                cardChildren.push(
                    h('div', { class: 'task-card-bottom' }, [
                        leftNode
                            ? h('div', { class: 'task-card-bottom-left' }, [
                                  leftNode,
                              ])
                            : null,
                        actions.length > 0
                            ? h('div', { class: 'task-card-actions' }, actions)
                            : null,
                    ]),
                )
            }

            return h(
                'div',
                {
                    class: ['task-card', { 'is-selected': checked }],
                    key: task.id,
                },
                cardChildren,
            )
        }

        return () => {
            if (props.tasks.length === 0) {
                return h('div', { class: 'task-card-empty' }, '暂无任务')
            }
            return h(
                'div',
                { class: 'task-card-list' },
                props.tasks.map((task) => renderTaskCard(task)),
            )
        }
    },
})
</script>

<style scoped>
.task-card-list {
    display: flex;
    flex-direction: column;
    gap: 6px;
    min-width: 0;
}

.task-card {
    min-width: 0;
    background: var(--bg-sidebar);
    border: 1px solid var(--border-color);
    border-radius: 12px;
    padding: 8px 10px;
    transition:
        border-color 0.2s,
        background-color 0.2s;
}

.task-card.is-selected {
    border-color: var(--color-primary);
    background: var(--color-primary-soft);
}

.task-card-top {
    display: flex;
    align-items: center;
    gap: 8px;
    min-width: 0;
}

.task-card-cover {
    width: 34px;
    height: 34px;
    border-radius: 7px;
    object-fit: cover;
    flex-shrink: 0;
}

.task-card-cover-fallback {
    display: grid;
    place-items: center;
    background: var(--color-primary-soft);
    color: var(--color-primary);
}

.task-card-song {
    flex: 1;
    min-width: 0;
}

.task-card-title {
    font-weight: 600;
    font-size: 13.5px;
    line-height: 1.3;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    color: var(--color-text);
}

.task-card-sub {
    font-size: 11.5px;
    line-height: 1.3;
    color: var(--color-text-secondary);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    margin-top: 2px;
}

.task-status-tag {
    flex-shrink: 0;
}

.task-card-bottom {
    display: flex;
    align-items: center;
    justify-content: space-between;
    flex-wrap: wrap;
    gap: 6px 8px;
    margin-top: 6px;
    padding-left: 26px;
    min-width: 0;
}

.task-card-bottom-left {
    flex: 1 1 140px;
    min-width: 0;
}

.task-card-progress-inline {
    display: flex;
    align-items: center;
    gap: 8px;
    min-width: 0;
}

.task-card-progress-inline :deep(.n-progress) {
    flex: 1;
    min-width: 48px;
}

.task-card-percent {
    flex-shrink: 0;
    font-size: 11px;
    color: var(--color-text-secondary);
    font-variant-numeric: tabular-nums;
}

.task-card-filepath {
    min-width: 0;
    overflow-wrap: anywhere;
    font-size: 11px;
    color: var(--color-text-secondary);
}

.task-card-status-message {
    font-size: 11.5px;
    line-height: 1.35;
    word-break: break-word;
    overflow-wrap: anywhere;
}

.task-card-status-message.is-error {
    color: var(--n-error-color, #ef4444);
}

.task-card-status-message.is-interrupted {
    color: var(--n-warning-color, #f59e0b);
}

.task-card-actions {
    display: flex;
    align-items: center;
    gap: 6px;
    flex-shrink: 0;
    margin-left: auto;
}

.task-card-actions :deep(.n-button) {
    height: 28px;
    min-height: 28px;
    padding: 0 10px;
    font-size: 12px;
    border-radius: 999px;
}

.task-card-empty {
    text-align: center;
    padding: 40px 0;
    color: var(--color-text-secondary);
    font-size: 14px;
}
</style>
