# JARVIS-
Just A Rather Very Intelligent
/**
 * Jarvis - Combined single-file implementation (client + server sections)
 *
 * Hinweis:
 * - Diese Datei enthält die gesamte Client-React-Logik (mehrere Komponenten),
 *   Utility-Funktionen und einen optionalen Express-Server-Endpoint als Kommentar/Section.
 * - In einem echten Projekt solltest du die Abschnitte in separate Dateien
 *   (z. B. src/components/*.tsx, src/utils/*.ts, server/*.js) splitten.
 *
 * Verwendung:
 * - Für die Frontend-Entwicklung: kopiere die Client-Abschnitte in ein React-Projekt (Vite/CRA).
 * - Für das Backend: die Express-Section kann in einem Node/Express-Server verwendet werden.
 *
 * Kopiere, teile oder extrahiere die Teile wie benötigt.
 */

/* ============================
   Imports (Client / React)
   ============================ */
import React, { useEffect, useRef, useState } from "react";

/* ============================
   Typen
   ============================ */
type Message = {
  id: string;
  role: "user" | "assistant";
  text: string;
  lang?: string;
};

/* ============================
   Utils: Speech (TTS)
   ============================ */
export async function speakText(text: string, lang = "de-DE") {
  return new Promise<void>((resolve) => {
    const synth = (window as any).speechSynthesis as SpeechSynthesis | undefined;
    if (!synth) {
      console.warn("SpeechSynthesis not supported");
      resolve();
      return;
    }

    // get voices; note: voices may be empty initially (async). Try to load if empty.
    let voices = synth.getVoices();
    if (!voices || voices.length === 0) {
      // try to wait briefly for voices to load
      voices = [];
      const loadHandler = () => {
        voices = synth.getVoices();
      };
      (window as any).speechSynthesis.onvoiceschanged = loadHandler;
      setTimeout(() => {
        voices = synth.getVoices();
      }, 250);
    }

    // heuristics to pick a male voice matching language
    const shortLang = lang.split("-")[0];
    const preferred = voices.find((v) => v.lang.startsWith(shortLang) && /male|Mark|David|Thomas|Hans|Joey|Paul|Daniel/i.test(v.name));
    let selected: SpeechSynthesisVoice | null = preferred || voices.find((v) => v.lang === lang) || voices[0] || null;

    const utter = new SpeechSynthesisUtterance(text);
    utter.lang = lang;
    if (selected) utter.voice = selected;
    // male + robotic characteristics
    utter.pitch = 0.45; // lower pitch
    // small variation so not always identical
    utter.rate = 0.85 + Math.random() * 0.2;
    utter.volume = 1.0;

    // resolve on end/error
    utter.onend = () => resolve();
    utter.onerror = (e) => {
      console.warn("TTS error", e);
      resolve();
    };

    synth.speak(utter);
  });
}

/* ============================
   Utils: API (client -> backend)
   ============================ */
export async function sendToJarvis(text: string, lang: string) {
  // default fetch to /api/jarvis; ensure your server provides this endpoint.
  const res = await fetch("/api/jarvis", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ text, lang }),
  });
  if (!res.ok) {
    const t = await res.text();
    throw new Error("API error: " + t);
  }
  const data = await res.json();
  // expected: { text: "...", lang: "de-DE" }
  return data;
}

/* ============================
   Component: MessageBubble
   ============================ */
export function MessageBubble({ role, text }: { role: "user" | "assistant"; text: string }) {
  const isUser = role === "user";
  return (
    <div style={{ display: "flex", justifyContent: isUser ? "flex-end" : "flex-start", margin: "8px 0" }}>
      <div
        style={{
          maxWidth: "70%",
          padding: 12,
          borderRadius: 12,
          background: isUser ? "#003a44" : "#00121a",
          color: "#fff",
          boxShadow: "0 2px 8px rgba(0,0,0,0.6)",
        }}
      >
        <div style={{ fontSize: 14, whiteSpace: "pre-wrap", lineHeight: 1.4 }}>{text}</div>
      </div>
    </div>
  );
}

/* ============================
   Component
