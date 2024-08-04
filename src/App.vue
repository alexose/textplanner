<template>
    <div id="app">
        <div class="container">
            <div class="pane left-pane">
                <textarea
                    v-if="isMarkdownView"
                    v-model="markdownContent"
                    class="editor-container"
                    placeholder="Enter Markdown content here..."
                ></textarea>
            </div>
            <div class="resizer" @mousedown="onMouseDown"></div>
            <div class="pane right-pane" @wheel="onWheel">
                <div class="day-view" ref="dayView">
                    <div class="hour" v-for="hour in 24" :key="hour" :style="{height: `${hourHeight}px`}">
                        {{ hour }}:00
                    </div>
                    <div class="task" v-for="task in tasks" :key="task.start.getTime()" :style="taskStyle(task)">
                        <div class="task-text">{{ task.text }}</div>
                    </div>
                    <div class="current-time-line" :style="currentTimeLineStyle"></div>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
    const chrono = require("chrono-node");

    export default {
        name: "App",
        data() {
            return {
                isMarkdownView: true,
                markdownContent: `# Simple Schedule

Prompt: "Today I need to do laundry, grind coffee, go to the hardware store (20 minute drive each way), exercise for 45 minutes, have lunch, and do 1.5 hours of reading. Could you make a schedule for me, breaking these tasks into subtasks if needed, and output as plain text?   Aim for about 15-20 tasks total.  Feel free to add have overlapping tasks where it makes sense." 

8:00 AM - 8:15 AM: Start laundry (Load 1)
8:00 AM - 8:15 AM: Grind coffee
8:15 AM - 8:35 AM: Drive to hardware store
8:35 AM - 9:05 AM: Shop at hardware store
9:05 AM - 9:25 AM: Drive back home
9:30 AM - 10:30 AM: Switch + Start Dryer
10:00 AM - 10:45 AM: Exercise
10:45 AM - 11:00 AM: Cool down and shower
11:00 AM - 11:30 AM: Have lunch
11:30 AM - 11:55 AM: Fold Laundry
12:30 PM - 1:00 PM: Reading session 1
1:00 PM - 1:15 PM: Break
1:15 PM - 1:45 PM: Reading session 2
1:45 PM - 2:00 PM: Break
2:00 PM - 2:30 PM: Reading session 3`,
                tasks: [],
                hourHeight: 160, // Initial height of each hour block
                currentTime: new Date(),
            };
        },
        computed: {
            currentTimeLineStyle() {
                const now = this.currentTime;
                const currentHour = now.getHours() + now.getMinutes() / 60;
                const top = currentHour * this.hourHeight;
                return {
                    top: `${top}px`,
                    height: "2px",
                    width: "100%",
                    backgroundColor: "blue",
                    position: "absolute",
                };
            },
        },
        watch: {
            markdownContent() {
                this.tasks = this.extractTasks(this.markdownContent);
            },
        },
        methods: {
            strip(text) {
                return text.replace(/^\s+|\s+$/g, "").replace(/^[\W_]+|[\W_]+$/g, "");
            },
            extractTasks(text) {
                let tasks = [];
                text.split("\n").forEach(line => {
                    const parsed = chrono.parse(line);
                    const item = parsed && parsed.length > 0 ? parsed[0] : null;
                    if (item && item.start && item.end && item.text) {
                        const start = item.start.date();
                        const end = item.end.date();
                        const remainingText = this.strip(line.replace(item.text, ""));
                        tasks.push({start, end, text: remainingText});
                    }
                });
                return this.arrangeTasks(tasks);
            },
            arrangeTasks(tasks) {
                const columns = [];
                tasks.forEach(task => {
                    let placed = false;
                    for (let i = 0; i < columns.length; i++) {
                        if (!this.isOverlap(columns[i], task)) {
                            columns[i].push(task);
                            placed = true;
                            break;
                        }
                    }
                    if (!placed) {
                        columns.push([task]);
                    }
                });
                return tasks.map(task => {
                    const column = columns.findIndex(col => col.includes(task));
                    return {...task, column};
                });
            },
            isOverlap(tasks, task) {
                return tasks.some(t => !(task.end <= t.start || task.start >= t.end));
            },
            toggleView() {
                this.isMarkdownView = !this.isMarkdownView;
            },
            onMouseDown(event) {
                const resizer = event.target;
                const leftPane = resizer.previousElementSibling;
                const rightPane = resizer.nextElementSibling;
                const container = resizer.parentElement;

                const onMouseMove = e => {
                    const containerRect = container.getBoundingClientRect();
                    const leftWidth = e.clientX - containerRect.left;
                    const rightWidth = containerRect.right - e.clientX;

                    if (leftWidth < 200 || rightWidth < 200) {
                        return;
                    }

                    leftPane.style.flexBasis = `${leftWidth}px`;
                    rightPane.style.flexBasis = `${rightWidth}px`;
                };

                const onMouseUp = () => {
                    document.removeEventListener("mousemove", onMouseMove);
                    document.removeEventListener("mouseup", onMouseUp);
                };

                document.addEventListener("mousemove", onMouseMove);
                document.addEventListener("mouseup", onMouseUp);
            },
            taskStyle(task) {
                const startHour = task.start.getHours() + task.start.getMinutes() / 60;
                const endHour = task.end.getHours() + task.end.getMinutes() / 60;
                const top = startHour * this.hourHeight;
                const height = (endHour - startHour) * this.hourHeight;
                const left = task.column * 220; // Adjust the width for each column

                return {
                    top: `${top}px`,
                    height: `${height}px`,
                    left: `${left}px`,
                };
            },
            onWheel(event) {
                if (!event.shiftKey) return; // Only zoom if SHIFT is pressed

                const dayView = this.$refs.dayView;
                const rect = dayView.getBoundingClientRect();
                const cursorY = event.clientY - rect.top;
                const scrollRatio = (cursorY + dayView.scrollTop) / dayView.scrollHeight;

                console.log(event.deltaY);

                if (event.deltaY < 0) {
                    // Zoom in
                    this.hourHeight = Math.min(this.hourHeight + 10, 200);
                } else {
                    // Zoom out
                    this.hourHeight = Math.max(this.hourHeight - 10, 20);
                }

                this.$nextTick(() => {
                    dayView.scrollTop = scrollRatio * dayView.scrollHeight - cursorY;
                });
            },
            updateCurrentTime() {
                this.currentTime = new Date();
            },
        },
        mounted() {
            this.tasks = this.extractTasks(this.markdownContent);
            // Scroll to the first task
            this.$nextTick(() => {
                const firstTask = this.$refs.dayView.querySelector(".task");
                if (firstTask) {
                    firstTask.scrollIntoView();
                }
            });
            this.updateCurrentTime();
            setInterval(this.updateCurrentTime, 60000); // Update every minute
        },
    };
</script>

<style>
    body {
        margin: 0;
        padding: 0;
        overflow: hidden;
    }

    #app {
        font-family: Avenir, Helvetica, Arial, sans-serif;
        -webkit-font-smoothing: antialiased;
        -moz-osx-font-smoothing: grayscale;
        margin: 0;
        padding: 0;
        height: 100vh;
    }

    .container {
        display: flex;
        width: 100%;
        height: 100vh;
        margin: 0;
        padding: 0;
        box-sizing: border-box;
    }

    .pane {
        flex: 1;
        overflow: auto;
        padding: 20px;
        box-sizing: border-box;
        border: 1px solid #ccc;
        overflow: hidden;
    }

    .left-pane {
        background-color: #fafafa;
    }

    .right-pane {
        background-color: #e0e0e0;
    }

    .resizer {
        width: 10px;
        cursor: ew-resize;
        background-color: #ccc;
    }

    .editor-container {
        width: 100%;
        height: 100%;
        padding: 20px;
        box-sizing: border-box;
        outline: none;
        border: none;
        background: none;
    }

    .day-view {
        position: relative;
        height: 100%;
        border: 1px solid #ccc;
        background-color: white;
        overflow-y: auto;
    }

    .hour {
        border-bottom: 1px solid #ddd;
        padding-left: 10px;
        box-sizing: border-box;
        color: #888;
    }

    .task {
        position: absolute;
        width: 200px;
        background-color: #f9f9f9;
        border: 1px solid #ccc;
        border-radius: 5px;
        padding: 5px;
        box-sizing: border-box;
        margin-left: 100px;
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
    }

    .current-time-line {
        position: absolute;
        width: 100%;
        height: 2px;
        background-color: blue;
        z-index: 10;
    }
</style>
