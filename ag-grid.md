Below is the Angular version of the same pattern, using:

autoGroupColumnDef

cellRenderer: 'agGroupCellRenderer' (keeps expand/collapse UI)

cellRendererParams.innerRenderer (your custom UI inside the group cell)

When the user changes the <select> on a group row, we update every leaf row in that group via node.allLeafChildren and setDataValue(...).

1) Inner renderer component (Angular)

group-status-inner-renderer.component.ts

import { Component } from '@angular/core';
import type { ICellRendererParams } from 'ag-grid-community';
import { ICellRendererAngularComp } from 'ag-grid-angular';

type Status = 'Pending' | 'Approved' | 'Rejected';

@Component({
  selector: 'app-group-status-inner-renderer',
  template: `
    <ng-container *ngIf="isGroup; else leafTpl">
      <span class="group-label">
        {{ params.value }} ({{ leafCount }})
      </span>

      <label class="group-edit">
        Group status:
        <select
          [ngModel]="currentGroupStatus"
          (ngModelChange)="onStatusChange($event)"
        >
          <option value="__mixed__" disabled>Mixed</option>
          <option value="Pending">Pending</option>
          <option value="Approved">Approved</option>
          <option value="Rejected">Rejected</option>
        </select>
      </label>
    </ng-container>

    <ng-template #leafTpl>
      <span>{{ params.value }}</span>
    </ng-template>
  `,
  styles: [`
    :host { display: inline-flex; align-items: center; gap: 12px; width: 100%; }
    .group-label { font-weight: 600; }
    .group-edit select { margin-left: 6px; }
  `],
})
export class GroupStatusInnerRendererComponent implements ICellRendererAngularComp {
  params!: ICellRendererParams;
  isGroup = false;
  leafCount = 0;
  currentGroupStatus: Status | '__mixed__' = '__mixed__';

  agInit(params: ICellRendererParams): void {
    this.params = params;
    this.recomputeFromNode();
  }

  refresh(params: ICellRendererParams): boolean {
    this.params = params;
    this.recomputeFromNode();
    return true;
  }

  private recomputeFromNode() {
    const node = this.params.node;
    this.isGroup = !!node.group;

    if (!this.isGroup) return;

    const leafChildren = node.allLeafChildren ?? [];
    this.leafCount = leafChildren.length;

    const statuses = new Set(
      leafChildren
        .map((child) => child.data?.status as Status | undefined)
        .filter(Boolean)
    );

    this.currentGroupStatus =
      statuses.size === 1 ? (Array.from(statuses)[0] as Status) : '__mixed__';
  }

  onStatusChange(newStatus: Status) {
    const node = this.params.node;
    const api = this.params.api;
    const leafChildren = node.allLeafChildren ?? [];

    leafChildren.forEach((child) => {
      child.setDataValue('status', newStatus);
    });

    // refresh the group cell so "Mixed"/value updates immediately
    api.refreshCells({ rowNodes: [node], force: true });
  }
}

Custom Angular cell components implement ICellRendererAngularComp with agInit/refresh.

2) Grid component using autoGroupColumnDef.innerRenderer

app.component.ts

import { Component } from '@angular/core';
import type { ColDef } from 'ag-grid-community';
import { GroupStatusInnerRendererComponent } from './group-status-inner-renderer.component';

type RowData = {
  id: number;
  department: string;
  employee: string;
  status: 'Pending' | 'Approved' | 'Rejected';
};

@Component({
  selector: 'my-app',
  template: `
    <ag-grid-angular
      class="ag-theme-alpine"
      style="width: 100%; height: 450px;"
      [rowData]="rowData"
      [columnDefs]="columnDefs"
      [autoGroupColumnDef]="autoGroupColumnDef"
      [groupDefaultExpanded]="1"
      [animateRows]="true"
      [getRowId]="getRowId"
    ></ag-grid-angular>
  `,
})
export class AppComponent {
  rowData: RowData[] = [
    { id: 1, department: 'Sales',   employee: 'Anna',  status: 'Pending' },
    { id: 2, department: 'Sales',   employee: 'Mark',  status: 'Pending' },
    { id: 3, department: 'Support', employee: 'Julia', status: 'Approved' },
    { id: 4, department: 'Support', employee: 'Tom',   status: 'Rejected' },
  ];

  columnDefs: ColDef<RowData>[] = [
    { field: 'department', rowGroup: true, hide: true },
    { field: 'employee' },
    { field: 'status', editable: true },
  ];

  // Keep default group cell renderer (chevron, indentation, etc),
  // but replace the "value part" with innerRenderer.
  autoGroupColumnDef: ColDef<RowData> = {
    headerName: 'Department',
    cellRenderer: 'agGroupCellRenderer',
    cellRendererParams: {
      suppressCount: false,
      innerRenderer: GroupStatusInnerRendererComponent,
    },
  };

  getRowId = (params: any) => String(params.data.id);
}

AG Grid documents autoGroupColumnDef.cellRendererParams.innerRenderer for customizing the display while keeping the built-in group renderer behavior.

3) Angular module setup (important)

Make sure AG Grid + FormsModule are included (because we used ngModel):

app.module.ts

import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { FormsModule } from '@angular/forms';
import { AgGridModule } from 'ag-grid-angular';

import { AppComponent } from './app.component';
import { GroupStatusInnerRendererComponent } from './group-status-inner-renderer.component';

@NgModule({
  declarations: [AppComponent, GroupStatusInnerRendererComponent],
  imports: [BrowserModule, FormsModule, AgGridModule],
  bootstrap: [AppComponent],
})
export class AppModule {}
Notes / gotchas

This is meant for row grouping (client-side). Group nodes expose allLeafChildren, which is what makes “apply to whole group” easy.

If you want to store a true “group-level” value (not just push to children), you’ll need a separate store keyed by group path.

If you tell me whether you’re grouping by one field or multiple fields, I can show how to generate a stable group key/path (like "Sales|2026"), so your group-level value persists even if rows change.
