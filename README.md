# Private
package main

import (
    "encoding/json"
    "fmt"
    "html/template"
    "log"
    "net/http"
    "strconv"
    "strings"
    "sync"
    "time"
)

type Task struct {
    ID          int       `json:"id"`
    Name        string    `json:"name"`
    Completed   bool      `json:"completed"`
    Created     time.Time `json:"created"`
}

type App struct {
    tasks []Task
    mu    sync.RWMutex
    nextID int
}

var tmpl = `
<!DOCTYPE html>
<html>
<head>
    <title>Task Manager</title>
    <style>
        body { font-family: Arial; max-width: 800px; margin: 50px auto; }
        input, button { padding: 10px; margin: 5px; }
        ul { list-style: none; padding: 0; }
        li { padding: 10px; border-bottom: 1px solid #ddd; display: flex; justify-content: space-between; align-items: center; }
        .completed { text-decoration: line-through; opacity: 0.6; }
        #score { font-size: 24px; font-weight: bold; color: #007bff; }
    </style>
</head>
<body>
    <h1>Task Manager</h1>
    <input type="text" id="taskInput" placeholder="Enter task name">
    <button onclick="addTask()">Add Task</button>
    <div>Auto Score: <span id="score">0</span></div>
    <ul id="taskList"></ul>

    <script>
        let tasks = [];
        
        function render() {
            const list = document.getElementById('taskList');
            const scoreEl = document.getElementById('score');
            list.innerHTML = '';
            let score = 0;
            
            tasks.forEach(task => {
                const li = document.createElement('li');
                li.className = task.completed ? 'completed' : '';
                li.innerHTML = `
                    <span>${task.name}</span>
                    <div>
                        <button onclick="toggleTask(${task.id})">${task.completed ? 'Undo' : 'Complete'}</button>
                    </div>
                `;
                list.appendChild(li);
                if (task.completed) score += 10;
            });
            scoreEl.textContent = score;
        }
        
        async function addTask() {
            const name = document.getElementById('taskInput').value.trim();
            if (!name) return;
            const resp = await fetch('/add', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({name: name})
            });
            location.reload();
        }
        
        async function toggleTask(id) {
            await fetch('/update', {
                method: 'POST',
                headers: {'Content-Type': 'application/json'},
                body: JSON.stringify({id: id})
            });
            location.reload();
        }
        
        async function loadTasks() {
            const resp = await fetch('/tasks');
            tasks = await resp.json();
            render();
        }
        
        loadTasks();
        setInterval(loadTasks, 2000);  // Auto-refresh score
    </script>
</body>
</html>`

func (a *App) handler(w http.ResponseWriter, r *http.Request) {
    tmpl.Execute(w, nil)
}

func (a *App) tasksHandler(w http.ResponseWriter, r *http.Request) {
    a.mu.RLock()
    tasks := make([]Task, len(a.tasks))
    copy(tasks, a.tasks)
    a.mu.RUnlock()
    json.NewEncoder(w).Encode(tasks)
}

func (a *App) addHandler(w http.ResponseWriter, r *http.Request) {
    var req struct{ Name string }
    json.NewDecoder(r.Body).Decode(&req)
    a.mu.Lock()
    a.tasks = append(a.tasks, Task{
        ID:        a.nextID,
        Name:      req.Name,
        Created:   time.Now(),
    })
    a.nextID++
    a.mu.Unlock()
    w.WriteHeader(http.StatusOK)
}

func (a *App) updateHandler(w http.ResponseWriter, r *http.Request) {
    var req struct{ ID int }
    json.NewDecoder(r.Body).Decode(&req)
    a.mu.Lock()
    for i := range a.tasks {
        if a.tasks[i].ID == req.ID {
            a.tasks[i].Completed = !a.tasks[i].Completed
            break
        }
    }
    a.mu.Unlock()
    w.WriteHeader(http.StatusOK)
}

func main() {
    app := &App{}
    http.HandleFunc("/", app.handler)
    http.HandleFunc("/tasks", app.tasksHandler)
    http.HandleFunc("/add", app.addHandler)
    http.HandleFunc("/update", app.updateHandler)
    fmt.Println("Server running on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
