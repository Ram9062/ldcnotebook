// Full PWA Entry Tracker App with PDF Export & Custom Install Prompt
import React, { useEffect, useState } from "react";
import { Button } from "@/components/ui/button";
import { Card, CardContent } from "@/components/ui/card";
import { Input } from "@/components/ui/input";
import { Trash2, Plus, RefreshCw, Download, MonitorDown } from "lucide-react";
import { format } from "date-fns";
import jsPDF from "jspdf";
import autoTable from "jspdf-autotable";

const getTodayDate = () => format(new Date(), "yyyy-MM-dd");

const generateTimeOptions = () => {
  const times = [];
  let hour = 0;
  let minute = 0;
  while (hour < 24) {
    const time = format(new Date(0, 0, 0, hour, minute), "hh:mm a");
    times.push({ label: time, value: format(new Date(0, 0, 0, hour, minute), "HH:mm") });
    minute += 30;
    if (minute === 60) {
      minute = 0;
      hour++;
    }
  }
  return times;
};

const TIME_OPTIONS = generateTimeOptions();
const LOCAL_KEY = "notebook-entries";

const App = () => {
  const [date, setDate] = useState(getTodayDate());
  const [entries, setEntries] = useState([]);
  const [showDeleteId, setShowDeleteId] = useState(null);
  const [deferredPrompt, setDeferredPrompt] = useState(null);
  const [showInstallPrompt, setShowInstallPrompt] = useState(false);

  useEffect(() => {
    const saved = JSON.parse(localStorage.getItem(LOCAL_KEY) || "[]");
    const filtered = saved.filter(e => {
      const entryDate = new Date(e.date);
      const sixMonthsAgo = new Date();
      sixMonthsAgo.setMonth(sixMonthsAgo.getMonth() - 6);
      return entryDate >= sixMonthsAgo;
    });
    setEntries(filtered);
    localStorage.setItem(LOCAL_KEY, JSON.stringify(filtered));
  }, []);

  useEffect(() => {
    const handler = (e) => {
      e.preventDefault();
      setDeferredPrompt(e);
      setShowInstallPrompt(true);
    };
    window.addEventListener("beforeinstallprompt", handler);
    return () => window.removeEventListener("beforeinstallprompt", handler);
  }, []);

  const handleInstall = async () => {
    if (deferredPrompt) {
      deferredPrompt.prompt();
      const { outcome } = await deferredPrompt.userChoice;
      if (outcome === "accepted") {
        setShowInstallPrompt(false);
      }
      setDeferredPrompt(null);
    }
  };

  const notebookEntries = entries.filter((e) => e.date === date);

  const saveEntries = (newEntries) => {
    setEntries(newEntries);
    localStorage.setItem(LOCAL_KEY, JSON.stringify(newEntries));
  };

  const addEntry = () => {
    const now = new Date();
    const time = format(now, "HH:mm");
    const newEntry = {
      id: Date.now().toString(),
      date,
      name: "",
      time,
      driver: "",
    };
    const updated = [...entries, newEntry];
    saveEntries(updated);
  };

  const updateEntry = (id, field, value) => {
    const updated = entries.map(e => e.id === id ? { ...e, [field]: value } : e);
    saveEntries(updated);
  };

  const deleteEntry = (id) => {
    const updated = entries.filter(e => e.id !== id);
    saveEntries(updated);
  };

  const handleDoubleClick = (id) => {
    setShowDeleteId(id);
  };

  const exportPDF = () => {
    const doc = new jsPDF();
    doc.text(`${date} Entries`, 14, 14);
    autoTable(doc, {
      head: [["Customer Name", "Time", "Driver Name"]],
      body: notebookEntries.map(e => [e.name, e.time, e.driver]),
    });
    doc.save(`${date}-entries.pdf`);
  };

  return (
    <main className="h-screen flex flex-col">
      <div className="sticky top-0 z-10 bg-white border-b shadow-sm">
        <Card>
          <CardContent className="p-4 flex flex-wrap gap-4 items-center justify-between">
            <h2 className="text-xl font-bold select-none pointer-events-none">Notebook</h2>
            <div className="select-none pointer-events-none">Total Entries: {notebookEntries.length}</div>
            <Input
              type="date"
              value={date}
              onChange={(e) => setDate(e.target.value)}
              className="max-w-[150px]"
            />
            <div className="flex gap-2 flex-wrap">
              <Button onClick={addEntry}>
                <Plus className="w-4 h-4 mr-2" /> Add Entry
              </Button>
              <Button onClick={exportPDF} variant="outline">
                <Download className="w-4 h-4 mr-2" /> Export PDF
              </Button>
            </div>
          </CardContent>
        </Card>
      </div>

      {showInstallPrompt && (
        <div className="fixed bottom-4 left-4 right-4 bg-white border rounded shadow-lg p-4 z-50 flex justify-between items-center gap-4">
          <div className="flex items-center gap-2">
            <MonitorDown className="text-blue-500 w-5 h-5" />
            <span>Install this app for quick access.</span>
          </div>
          <div className="flex gap-2">
            <Button onClick={handleInstall}>Install</Button>
            <Button variant="outline" onClick={() => setShowInstallPrompt(false)}>Not Now</Button>
          </div>
        </div>
      )}

      <div className="overflow-y-auto flex-1 p-4 space-y-4">
        {notebookEntries.map((entry) => (
          <Card
            key={entry.id}
            onDoubleClick={() => handleDoubleClick(entry.id)}
            className="relative"
          >
            <CardContent className="p-4 grid grid-cols-[1fr_auto] gap-4 items-center">
              <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
                <Input
                  placeholder="Customer Name"
                  value={entry.name}
                  onChange={(e) => updateEntry(entry.id, "name", e.target.value)}
                />
                <select
                  className="border rounded px-2 py-1"
                  value={entry.time}
                  onChange={(e) => updateEntry(entry.id, "time", e.target.value)}
                >
                  {TIME_OPTIONS.map((opt) => (
                    <option key={opt.value} value={opt.value}>
                      {opt.label}
                    </option>
                  ))}
                </select>
                <Input
                  placeholder="Driver Name"
                  value={entry.driver}
                  onChange={(e) => updateEntry(entry.id, "driver", e.target.value)}
                />
              </div>
              {showDeleteId === entry.id && (
                <Button
                  variant="destructive"
                  onClick={() => deleteEntry(entry.id)}
                >
                  <Trash2 className="w-4 h-4" />
                </Button>
              )}
            </CardContent>
          </Card>
        ))}
      </div>
    </main>
  );
};

export default App;
