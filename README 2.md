Entregable 2 Modo offline
Servicio de detección de red en tiempo real código

import { Injectable } from '@angular/core';
import { Network } from '@capacitor/network';
import { BehaviorSubject } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class NetworkService {
  private onlineSubject = new BehaviorSubject<boolean>(true);
  public isOnline$ = this.onlineSubject.asObservable();
  public isOnline = true;

  async init() {
    const status = await Network.getStatus();
    this.isOnline = status.connected;
    this.onlineSubject.next(status.connected);

    Network.addListener('networkStatusChange', status => {
      console.log('Estado de red:', status.connected);
      this.isOnline = status.connected;
      this.onlineSubject.next(status.connected);
    });
  }
}




Servicio de almacenamiento local y sincronización automática
import { Injectable, inject } from '@angular/core';
import { NetworkService } from './network.service';
import { Firestore, collection, addDoc } from '@angular/fire/firestore';
import { Auth } from '@angular/fire/auth';
import { ToastController } from '@ionic/angular/standalone';

interface PendingOperation {
  id: string;
  coleccion: string;
  datos: any;
  timestamp: number;
}

@Injectable({ providedIn: 'root' })
export class OfflineService {
  private networkService = inject(NetworkService);
  private firestore = inject(Firestore);
  private auth = inject(Auth);
  private toastCtrl = inject(ToastController);

  private STORAGE_KEY = 'medicalert_pending_ops';

  getPendingOps(): PendingOperation[] {
    const data = localStorage.getItem(this.STORAGE_KEY);
    return data ? JSON.parse(data) : [];
  }

  private savePendingOps(ops: PendingOperation[]) {
    localStorage.setItem(this.STORAGE_KEY, JSON.stringify(ops));
  }

  async guardarDato(coleccion: string, datos: any): Promise<boolean> {
    if (this.networkService.isOnline) {
      try {
        const uid = this.auth.currentUser?.uid;
        const ref = collection(this.firestore, `usuarios/${uid}/${coleccion}`);
        await addDoc(ref, { ...datos, fecha: new Date() });
        return true;
      } catch (e) {
        await this.guardarLocalmente(coleccion, datos);
        return false;
      }
    } else {
      await this.guardarLocalmente(coleccion, datos);
      return false;
    }
  }

  private async guardarLocalmente(coleccion: string, datos: any) {
    const ops = this.getPendingOps();
    const nueva: PendingOperation = {
      id: Date.now().toString(),
      coleccion,
      datos: { ...datos, fecha: new Date().toISOString() },
      timestamp: Date.now()
    };
    ops.push(nueva);
    this.savePendingOps(ops);

    const toast = await this.toastCtrl.create({

      mensaje: 'Sin conexion. Dato guardado localmente. Se sincronizará cuando recuperes la red.',
      duration: 3500,
      color: 'warning',
      position: 'bottom'
    });
    await toast.present();
  }

  async sincronizarPendientes(): Promise<number> {
    if (!this.networkService.isOnline) return 0;

    const ops = this.getPendingOps();
    if (ops.length === 0) return 0;

    let sincronizados = 0;
    const fallidos: PendingOperation[] = [];

    for (const op of ops) {
      try {
        const uid = this.auth.currentUser?.uid;
        const ref = collection(this.firestore, `usuarios/${uid}/${op.coleccion}`);
        await addDoc(ref, op.datos);
        sincronizados++;
      } catch (e) {
        fallidos.push(op);
      }
    }

    this.savePendingOps(fallidos);

    if (sincronizados > 0) {
      const toast = await this.toastCtrl.create({
        message: `${sincronizados} dato(s) sincronizado(s) con el servidor`,
        duration: 3000,
        color: 'success',
        position: 'bottom'
      });
      await toast.present();
    }

    return sincronizados;
  }

  getPendingCount(): number {
    return this.getPendingOps().length;
  }
}


Componente visual de estado de conectividad
import { Component, inject, OnInit, OnDestroy } from '@angular/core';
import { CommonModule } from '@angular/common';
import { IonIcon } from '@ionic/angular/standalone';
import { NetworkService } from '../../services/network.service';
import { OfflineService } from '../../services/offline.service';
import { addIcons } from 'ionicons';
import { cloudOfflineOutline, cloudDoneOutline, syncOutline } from 'ionicons/icons';
import { Subscription } from 'rxjs';

@Component({
  selector: 'app-network-banner',
  template: `
    <div class="network-banner offline" *ngIf="!isOnline">
      <ion-icon name="cloud-offline-outline"></ion-icon>
      <span>Sin conexion a internet</span>
      <div class="pending-badge" *ngIf="pendingCount > 0">
        {{ pendingCount }} pendiente(s)
      </div>
    </div>
    <div class="network-banner online" *ngIf="isOnline && showOnline">
      <ion-icon name="cloud-done-outline"></ion-icon>
      <span>Conexion restaurada</span>
      <span class="sync-text" *ngIf="sincronizando">Sincronizando...</span>
    </div>
  `,
  styles: [`
    .network-banner {
      display: flex;
      align-items: center;
      gap: 8px;
      padding: 10px 16px;
      font-size: 13px;
      font-weight: 500;
      z-index: 9999;
      ion-icon { font-size: 18px; }
      &.offline { background: #b71c1c; color: white; }
      &.online { background: #1b5e20; color: white; }
      .pending-badge {
        margin-left: auto;
        background: rgba(255,255,255,0.25);
        padding: 2px 8px;
        border-radius: 20px;
        font-size: 11px;
      }
      .sync-text { margin-left: auto; font-size: 11px; opacity: 0.8; }
    }
  `],
  standalone: true,
  imports: [CommonModule, IonIcon]
})
export class NetworkBannerComponent implements OnInit, OnDestroy {
  private networkService = inject(NetworkService);
  private offlineService = inject(OfflineService);

  isOnline = true;
  showOnline = false;
  sincronizando = false;
  pendingCount = 0;
  private sub!: Subscription;

  constructor() {
    addIcons({ cloudOfflineOutline, cloudDoneOutline, syncOutline });
  }

  ngOnInit() {
    this.isOnline = this.networkService.isOnline;
    this.pendingCount = this.offlineService.getPendingCount();

    this.sub = this.networkService.isOnline$.subscribe(async online => {
      const wasOffline = !this.isOnline;
      this.isOnline = online;
      this.pendingCount = this.offlineService.getPendingCount();

      if (online && wasOffline) {
        this.showOnline = true;
        this.sincronizando = true;
        await this.offlineService.sincronizarPendientes();
        this.sincronizando = false;
        this.pendingCount = this.offlineService.getPendingCount();
        setTimeout(() => { this.showOnline = false; }, 4000);
      }
    });
  }

  ngOnDestroy() {
    if (this.sub) this.sub.unsubscribe();
  }
}

Integración del OfflineService en el formulario de medicamentos
import { Component, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { Router } from '@angular/router';
import { OfflineService } from '../../services/offline.service';
import {
  IonContent, IonIcon, IonInput, IonItem, IonSelect,
  IonSelectOption, ToastController, LoadingController
} from '@ionic/angular/standalone';
import { addIcons } from 'ionicons';
import { arrowBackOutline, addOutline, timeOutline, trashOutline } from 'ionicons/icons';

@Component({
  selector: 'app-agregar-medicamento',
  templateUrl: './agregar-medicamento.page.html',
  styleUrls: ['./agregar-medicamento.page.scss'],
  standalone: true,
  imports: [CommonModule, FormsModule, IonContent, IonIcon,
    IonInput, IonItem, IonSelect, IonSelectOption]
})
export class AgregarMedicamentoPage {
  private offlineService = inject(OfflineService);
  private router = inject(Router);
  private toastCtrl = inject(ToastController);
  private loadingCtrl = inject(LoadingController);

  nombre = '';
  dosis = '';
  categoria = 'otro';
  frecuencia = '12h';
  duracionDias = 30;
  horarios: string[] = ['08:00'];
  nuevoHorario = '';

  categorias = [
    { valor: 'cardiovascular', label: 'Cardiovascular' },
    { valor: 'diabetes', label: 'Diabetes' },
    { valor: 'dolor', label: 'Dolor / Inflamacion' },
    { valor: 'vitaminas', label: 'Vitaminas / Suplementos' },
    { valor: 'otro', label: 'Otro' }
  ];

  frecuencias = [
    { valor: '8h', label: 'Cada 8 horas' },
    { valor: '12h', label: 'Cada 12 horas' },
    { valor: '24h', label: 'Una vez al dia' },
    { valor: 'semanal', label: 'Semanal' }
  ];

  constructor() {
    addIcons({ arrowBackOutline, addOutline, timeOutline, trashOutline });
  }

  agregarHorario() {
    if (this.nuevoHorario && !this.horarios.includes(this.nuevoHorario)) {
      this.horarios.push(this.nuevoHorario);
      this.nuevoHorario = '';
    }
  }

  eliminarHorario(h: string) {
    this.horarios = this.horarios.filter(x => x !== h);
  }

  async guardar() {
    if (!this.nombre || !this.dosis || this.horarios.length === 0) {
      const toast = await this.toastCtrl.create({
        message: 'Completa nombre, dosis y al menos un horario',
        duration: 2500, color: 'warning'
      });
      await toast.present();
      return;
    }

    const loading = await this.loadingCtrl.create({ message: 'Guardando...' });
    await loading.present();

    const medicamento = {
      nombre: this.nombre,
      dosis: this.dosis,
      categoria: this.categoria,
      frecuencia: this.frecuencia,
      horarios: this.horarios,
      duracionDias: this.duracionDias,
      activo: true
    };

    const guardado = await this.offlineService.guardarDato('medicamentos', medicamento);
    await loading.dismiss();

    const toast = await this.toastCtrl.create({
      message: guardado
        ? `${this.nombre} agregado correctamente`
        : `${this.nombre} guardado localmente. Se sincronizara al recuperar la red.`,
      duration: 3000,
      color: guardado ? 'success' : 'warning'
    });
    await toast.present();
    this.router.navigate(['/medicamentos'], { replaceUrl: true });
  }

  volver() { this.router.navigate(['/medicamentos']); }
}
Integración del banner en la aplicación principal
<ion-router-outlet id="main-content">
  <app-network-banner></app-network-banner>
</ion-router-outlet>
nicialización del NetworkService al arrancar la app
constructor() {
  addIcons({ ... });
  this.networkService.init();
}
