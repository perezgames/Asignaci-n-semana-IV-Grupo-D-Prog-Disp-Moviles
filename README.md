# Asignación-semana-IV-Grupo-D-Prog-Disp-Moviles
Repositorio de la asignación de la semana IV del Facilitador Joan Gregorio Pérez 

Este repositorio contiene el desarrollo de la Unidad 4 de Proyección Dispositivos Móviles, enfocado en la implementación de un módulo de conectividad para aplicaciones móviles híbridas con Ionic y Capacitor.
El proyecto está dividido en dos partes: un detector de estado de red que indica en tiempo real si el dispositivo tiene conexión a internet, y un modo offline funcional integrado en la app MedicAlert RD, que permite guardar datos localmente cuando no hay conexión y sincronizarlos automáticamente con Firebase Firestore al recuperarla.
Integrantes: Jeancarlos Almonte Padilla · Stuart Brandon Capellán De Los Santos · Jan Michael Perez Feliz


**Entregable 1 Detector red**
network.service.ts
import { Injectable } from '@angular/core';
import { BehaviorSubject } from 'rxjs';
import { Network } from '@capacitor/network';

@Injectable({ providedIn: 'root' })
export class NetworkService {
  private onlineStatus = new BehaviorSubject<boolean>(true);

  constructor() {
    this.initializeNetworkListener();
  }

  async initializeNetworkListener() {
    const status = await Network.getStatus();
    this.onlineStatus.next(status.connected);

    Network.addListener('networkStatusChange', (status) => {
      console.log('Estado de red:', status.connected);
      this.onlineStatus.next(status.connected);
    });
  }

  getNetworkStatus() {
    return this.onlineStatus.asObservable();
  }
}
network-status.component.ts
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { IonicModule } from '@ionic/angular';
import { NetworkService } from '../services/network.service';



@Component({
  selector: 'app-network-status',
  templateUrl: './network-status.component.html',
  styleUrls: ['./network-status.component.scss'],
  standalone: true,
  imports: [CommonModule, IonicModule]
})
export class NetworkStatusComponent implements OnInit {
  isOnline = true;

  constructor(private networkService: NetworkService) {}

  ngOnInit() {
    this.networkService.getNetworkStatus()
  . subscribe (status => this.isOnline = status);
  }
}
network-status.component.html
<ion-chip color="success" *ngIf="isOnline">
  <ion-label>Online</ion-label>
</ion-chip>

<ion-chip color="danger" *ngIf="!isOnline">
  <ion-label>Offline</ion-label>
</ion-chip>
home.page.ts
imports: [
  IonHeader,
  IonToolbar,
  IonTitle,
  IonContent,
  NetworkStatusComponent
]


app.component.ts
imports: [
  IonApp,
  IonRouterOutlet,
  NetworkStatusComponent
]
app.component.html
<ion-app>
  <app-network-status></app-network-status>
  <ion-router-outlet></ion-router-outlet>
</ion-app>


