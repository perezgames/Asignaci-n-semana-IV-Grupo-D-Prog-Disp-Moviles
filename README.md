# Asignación-semana-IV-Grupo-D-Prog-Disp-Moviles
Repositorio de la asignación de la semana IV del Facilitador Joan Gregorio Pérez 

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


