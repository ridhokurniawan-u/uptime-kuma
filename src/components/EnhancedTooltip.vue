<template>
    <div
        v-if="visible && content"
        class="custom-tooltip"
        :class="[positionClass, { 'dark': $root.isDark }]"
        :style="tooltipStyle"
    >
        <div class="tooltip-content">
            <!-- Status with colored indicator -->
            <div class="tooltip-row status-row">
                <span class="tooltip-label">Status:</span>
                <span class="status-indicator" :class="statusClass">
                    <i :class="statusIcon"></i>
                    {{ statusText }}
                </span>
            </div>
            
            <!-- Date/Time -->
            <div class="tooltip-row">
                <span class="tooltip-label">Time:</span>
                <span class="tooltip-value">{{ formatDate(content.time) }}</span>
            </div>
            
            <!-- Response Time (if available) -->
            <div v-if="hasResponseTime" class="tooltip-row">
                <span class="tooltip-label">Response:</span>
                <span class="tooltip-value response-time">{{ responseTime }}ms</span>
            </div>
            
            <!-- Response Code (if available) -->
            <div v-if="hasResponseCode" class="tooltip-row">
                <span class="tooltip-label">Code:</span>
                <span class="tooltip-value http-code" :class="httpStatusClass">{{ responseCode }}</span>
            </div>
            
            <!-- Total Uptime Percentage -->
            <div v-if="uptimePercentage !== null" class="tooltip-row uptime-row">
                <span class="tooltip-label">Uptime:</span>
                <span class="tooltip-value uptime-percentage" :class="uptimeClass">
                    <i class="fas fa-chart-line"></i>
                    {{ uptimePercentage }}%
                </span>
            </div>
            
            <!-- Error message (if available) -->
            <div v-if="hasErrorMessage" class="tooltip-row error-row">
                <span class="tooltip-label">Error:</span>
                <span class="tooltip-value error-msg">{{ content.msg }}</span>
            </div>
        </div>
        
        <!-- Tooltip arrow -->
        <div class="tooltip-arrow" :class="arrowPosition"></div>
    </div>
</template>

<script>
import dayjs from "dayjs";
import { DOWN, UP, PENDING, MAINTENANCE } from "../util.ts";

export default {
    name: "EnhancedTooltip",
    props: {
        visible: {
            type: Boolean,
            default: false
        },
        content: {
            type: Object,
            default: null
        },
        x: {
            type: Number,
            default: 0
        },
        y: {
            type: Number,
            default: 0
        },
        position: {
            type: String,
            default: "below" // "above" or "below"
        },
        monitorId: {
            type: Number,
            default: null
        }
    },
    computed: {
        tooltipStyle() {
            return {
                left: `${this.x}px`,
                top: `${this.y}px`,
                transform: 'translateX(-50%)',
                zIndex: 9999
            };
        },
        
        positionClass() {
            return `tooltip-${this.position}`;
        },
        
        arrowPosition() {
            return this.position === "above" ? "arrow-down" : "arrow-up";
        },
        
        statusText() {
            if (!this.content) return "Unknown";
            
            switch (this.content.status) {
                case UP:
                    return "Up";
                case DOWN:
                    return "Down";
                case PENDING:
                    return "Pending";
                case MAINTENANCE:
                    return "Maintenance";
                default:
                    return "Unknown";
            }
        },
        
        statusClass() {
            if (!this.content) return "";
            
            switch (this.content.status) {
                case UP:
                    return "status-up";
                case DOWN:
                    return "status-down";
                case PENDING:
                    return "status-pending";
                case MAINTENANCE:
                    return "status-maintenance";
                default:
                    return "status-unknown";
            }
        },
        
        statusIcon() {
            if (!this.content) return "fas fa-question";
            
            switch (this.content.status) {
                case UP:
                    return "fas fa-check-circle";
                case DOWN:
                    return "fas fa-times-circle";
                case PENDING:
                    return "fas fa-clock";
                case MAINTENANCE:
                    return "fas fa-wrench";
                default:
                    return "fas fa-question";
            }
        },
        
        // Check if response time is available and valid
        hasResponseTime() {
            if (!this.content) return false;
            const ping = this.responseTime;
            return ping !== undefined && ping !== null && !isNaN(ping) && ping >= 0;
        },
        
        // Get response time from various possible fields
        responseTime() {
            if (!this.content) return null;
            return this.content.ping ?? this.content.responseTime ?? this.content.response_time ?? null;
        },
        
        // Check if response code is available
        hasResponseCode() {
            return this.responseCode !== null && this.responseCode !== undefined;
        },
        
        responseCode() {
            if (!this.content) return null;
            
            // First try the direct fields
            const directCode = this.content.status_code ?? 
                              this.content.responseCode ?? 
                              this.content.code ?? 
                              this.content.http_code ?? 
                              this.content.httpCode ?? 
                              null;
            
            if (directCode !== null) return directCode;
            
            // Try to extract from message field (e.g., "200 - OK")
            if (this.content.msg) {
                const match = this.content.msg.match(/^(\d{3})\s*-/);
                if (match) {
                    return parseInt(match[1]);
                }
            }
            
            return null;
        },
        
        // Check if error message should be shown
        hasErrorMessage() {
            return this.content?.msg && 
                   (this.content.status === DOWN || this.content.status === 0);
        },
        
        uptimePercentage() {
            // Try multiple ways to get uptime data
            if (!this.monitorId) return null;
            
            // Method 1: From $root.uptimeList
            if (this.$root.uptimeList && this.$root.uptimeList[this.monitorId]) {
                const uptimeData = this.$root.uptimeList[this.monitorId];
                const uptime = uptimeData["24"] ?? 
                              uptimeData["1"] ??
                              uptimeData.uptime ??
                              uptimeData["24h"] ??
                              uptimeData["1d"];
                if (uptime !== undefined && uptime !== null) {
                    return Math.round(uptime * 10000) / 100;
                }
            }
            
            // Method 2: From monitor data
            if (this.$root.monitorList && this.$root.monitorList[this.monitorId]) {
                const monitor = this.$root.monitorList[this.monitorId];
                if (monitor.uptime !== undefined && monitor.uptime !== null) {
                    return Math.round(monitor.uptime * 10000) / 100;
                }
            }
            
            // Method 3: From stats data (if available in root)
            if (this.$root.stats && this.$root.stats[this.monitorId]) {
                const stats = this.$root.stats[this.monitorId];
                if (stats.uptime !== undefined) {
                    return Math.round(stats.uptime * 100) / 100;
                }
            }
            
            // Method 4: From content itself (if it contains uptime info)
            if (this.content && this.content.uptime !== undefined) {
                return Math.round(this.content.uptime * 10000) / 100;
            }
            
            // Method 5: Try to find uptime from any root property containing uptime
            try {
                for (const key in this.$root) {
                    if (typeof this.$root[key] === 'object' && this.$root[key] && this.$root[key][this.monitorId]) {
                        const data = this.$root[key][this.monitorId];
                        if (data.uptime !== undefined || data['24'] !== undefined) {
                            const uptime = data.uptime ?? data['24'];
                            if (uptime !== null && uptime !== undefined) {
                                return Math.round(uptime * 10000) / 100;
                            }
                        }
                    }
                }
            } catch (e) {
                // Ignore errors in exploration
            }
            
            return null;
        },
        
        uptimeClass() {
            if (this.uptimePercentage === null) return "";
            
            if (this.uptimePercentage >= 99.5) return "uptime-excellent";
            if (this.uptimePercentage >= 95) return "uptime-good";
            if (this.uptimePercentage >= 90) return "uptime-fair";
            return "uptime-poor";
        },
        
        httpStatusClass() {
            const code = this.responseCode;
            if (!code) return "";
            
            if (code >= 200 && code < 300) return "http-success";
            if (code >= 300 && code < 400) return "http-redirect";
            if (code >= 400 && code < 500) return "http-client-error";
            if (code >= 500) return "http-server-error";
            return "";
        }
    },
    methods: {
        formatDate(dateString) {
            if (!dateString) return "Unknown";
            return dayjs(dateString).format("MMM DD, HH:mm:ss");
        }
    }
};
</script>

<style lang="scss" scoped>
@import "../assets/vars.scss";

.custom-tooltip {
    position: fixed;
    background: rgba(0, 0, 0, 0.9);
    color: white;
    padding: 12px;
    border-radius: 8px;
    font-size: 12px;
    min-width: 200px;
    max-width: 300px;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
    backdrop-filter: blur(10px);
    border: 1px solid rgba(255, 255, 255, 0.1);
    
    &.dark {
        background: rgba(255, 255, 255, 0.95);
        color: #333;
        border: 1px solid rgba(0, 0, 0, 0.1);
    }
}

.tooltip-content {
    display: flex;
    flex-direction: column;
    gap: 6px;
}

.tooltip-row {
    display: flex;
    justify-content: space-between;
    align-items: center;
    
    &.status-row {
        border-bottom: 1px solid rgba(255, 255, 255, 0.2);
        padding-bottom: 6px;
        margin-bottom: 2px;
        
        .dark & {
            border-bottom-color: rgba(0, 0, 0, 0.2);
        }
    }
    
    &.error-row, &.uptime-row {
        border-top: 1px solid rgba(255, 255, 255, 0.2);
        padding-top: 6px;
        margin-top: 2px;
        
        .dark & {
            border-top-color: rgba(0, 0, 0, 0.2);
        }
    }
}

.tooltip-label {
    font-weight: 600;
    margin-right: 8px;
    opacity: 0.8;
    min-width: 60px;
}

.tooltip-value {
    font-weight: 500;
    text-align: right;
}

.status-indicator {
    display: flex;
    align-items: center;
    gap: 6px;
    font-weight: 600;
    
    i {
        font-size: 14px;
    }
    
    &.status-up {
        color: #28a745;
    }
    
    &.status-down {
        color: #dc3545;
    }
    
    &.status-pending {
        color: #ffc107;
    }
    
    &.status-maintenance {
        color: #6f42c1;
    }
    
    &.status-unknown {
        color: #6c757d;
    }
}

.response-time {
    color: #17a2b8;
    font-family: monospace;
}

.http-code {
    font-family: monospace;
    padding: 2px 6px;
    border-radius: 4px;
    font-size: 11px;
    
    &.http-success {
        background-color: #28a745;
        color: white;
    }
    
    &.http-redirect {
        background-color: #17a2b8;
        color: white;
    }
    
    &.http-client-error {
        background-color: #ffc107;
        color: #212529;
    }
    
    &.http-server-error {
        background-color: #dc3545;
        color: white;
    }
}

.uptime-percentage {
    display: flex;
    align-items: center;
    gap: 4px;
    font-weight: 600;
    
    i {
        font-size: 12px;
    }
    
    &.uptime-excellent {
        color: #28a745;
    }
    
    &.uptime-good {
        color: #17a2b8;
    }
    
    &.uptime-fair {
        color: #ffc107;
    }
    
    &.uptime-poor {
        color: #dc3545;
    }
}

.error-msg {
    color: #dc3545;
    font-size: 11px;
    max-width: 150px;
    word-break: break-word;
    text-align: right;
    
    .dark & {
        color: #dc3545;
    }
}

.tooltip-arrow {
    position: absolute;
    width: 0;
    height: 0;
    
    &.arrow-up {
        top: -8px;
        left: 50%;
        transform: translateX(-50%);
        border-left: 8px solid transparent;
        border-right: 8px solid transparent;
        border-bottom: 8px solid rgba(0, 0, 0, 0.9);
        
        .dark & {
            border-bottom-color: rgba(255, 255, 255, 0.95);
        }
    }
    
    &.arrow-down {
        bottom: -8px;
        left: 50%;
        transform: translateX(-50%);
        border-left: 8px solid transparent;
        border-right: 8px solid transparent;
        border-top: 8px solid rgba(0, 0, 0, 0.9);
        
        .dark & {
            border-top-color: rgba(255, 255, 255, 0.95);
        }
    }
}

// Animation
.custom-tooltip {
    animation: tooltipFadeIn 0.2s ease-out;
}

@keyframes tooltipFadeIn {
    from {
        opacity: 0;
        transform: translateX(-50%) translateY(-5px);
    }
    to {
        opacity: 1;
        transform: translateX(-50%) translateY(0);
    }
}
</style>