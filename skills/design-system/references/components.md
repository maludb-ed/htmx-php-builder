# Duralux component patterns (exact markup, "nxl-" template)

All snippets are copied verbatim from the template (class names and nesting preserved), trimmed
to their minimal repeating unit. Page content always lives inside
`main.nxl-container > .nxl-content > .main-content > .row > .col-* > .card`.

## Page header / breadcrumb / title with action buttons

Usage: first child of `.nxl-content`, before `.main-content`. Put action buttons inside
`.page-header-right-items-wrapper`. From customers.html.

```html
<div class="page-header">
    <div class="page-header-left d-flex align-items-center">
        <div class="page-header-title">
            <h5 class="m-b-10">Customers</h5>
        </div>
        <ul class="breadcrumb">
            <li class="breadcrumb-item"><a href="index.html">Home</a></li>
            <li class="breadcrumb-item">Customers</li>
        </ul>
    </div>
    <div class="page-header-right ms-auto">
        <div class="page-header-right-items">
            <div class="d-flex d-md-none">
                <a href="javascript:void(0)" class="page-header-right-close-toggle">
                    <i class="feather-arrow-left me-2"></i>
                    <span>Back</span>
                </a>
            </div>
            <div class="d-flex align-items-center gap-2 page-header-right-items-wrapper">
                <!-- icon button that toggles an optional stats strip (see below) -->
                <a href="javascript:void(0);" class="btn btn-icon btn-light-brand" data-bs-toggle="collapse" data-bs-target="#collapseOne">
                    <i class="feather-bar-chart"></i>
                </a>
                <!-- icon dropdown button (filter / export menus) -->
                <div class="dropdown">
                    <a class="btn btn-icon btn-light-brand" data-bs-toggle="dropdown" data-bs-offset="0, 10" data-bs-auto-close="outside">
                        <i class="feather-filter"></i>
                    </a>
                    <div class="dropdown-menu dropdown-menu-end">
                        <a href="javascript:void(0);" class="dropdown-item">
                            <i class="feather-eye me-3"></i>
                            <span>All</span>
                        </a>
                        <!-- repeat .dropdown-item; use <div class="dropdown-divider"></div> between groups -->
                    </div>
                </div>
                <!-- primary page action -->
                <a href="customers-create.html" class="btn btn-primary">
                    <i class="feather-plus me-2"></i>
                    <span>Create Customer</span>
                </a>
            </div>
        </div>
        <div class="d-md-none d-flex align-items-center">
            <a href="javascript:void(0)" class="page-header-right-open-toggle">
                <i class="feather-align-right fs-20"></i>
            </a>
        </div>
    </div>
</div>
```

Optional collapsible KPI strip directly after `.page-header` (toggled by the `feather-bar-chart`
button above; one card per stat, 4 across):

```html
<div id="collapseOne" class="accordion-collapse collapse page-header-collapse">
    <div class="accordion-body pb-2">
        <div class="row">
            <div class="col-xxl-3 col-md-6">
                <div class="card stretch stretch-full">
                    <div class="card-body">
                        <div class="d-flex align-items-center justify-content-between">
                            <div class="d-flex align-items-center gap-3">
                                <div class="avatar-text avatar-xl rounded">
                                    <i class="feather-users"></i>
                                </div>
                                <a href="javascript:void(0);" class="fw-bold d-block">
                                    <span class="text-truncate-1-line">Total Customers</span>
                                    <span class="fs-24 fw-bolder d-block">26,595</span>
                                </a>
                            </div>
                            <div class="badge bg-soft-success text-success">
                                <i class="feather-arrow-up fs-10 me-1"></i>
                                <span>36.85%</span>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
            <!-- repeat .col-xxl-3.col-md-6 -->
        </div>
    </div>
</div>
```

For form pages the action area holds Save/Cancel instead (customers-create.html) — note the
**save buttons live in the page header, not at the bottom of the form**:

```html
<div class="d-flex align-items-center gap-2 page-header-right-items-wrapper">
    <a href="javascript:void(0);" class="btn btn-light-brand successAlertMessage">
        <i class="feather-layers me-2"></i>
        <span>Save as Draft</span>
    </a>
    <a href="javascript:void(0);" class="btn btn-primary successAlertMessage">
        <i class="feather-user-plus me-2"></i>
        <span>Create Customer</span>
    </a>
</div>
```

## Card container pattern

Usage: every content block is a card inside a grid column. `stretch stretch-full` makes cards in
a row equal height. Card header title + kebab dropdown from customers-view.html.

```html
<div class="row">
    <div class="col-lg-12">
        <div class="card stretch stretch-full">
            <div class="card-header">
                <h5 class="card-title">Payment Record</h5>
                <div class="dropdown">
                    <a href="javascript:void(0);" class="avatar-text avatar-sm" data-bs-toggle="dropdown" data-bs-offset="25,25">
                        <i class="feather feather-more-vertical"></i>
                    </a>
                    <ul class="dropdown-menu dropdown-menu-end">
                        <li>
                            <a href="javascript:void(0);" class="dropdown-item">
                                <i class="feather feather-edit-3 me-3"></i>
                                <span>Edit</span>
                            </a>
                        </li>
                    </ul>
                </div>
            </div>
            <div class="card-body">
                <!-- content -->
            </div>
        </div>
    </div>
</div>
```

Optional window-style header controls (index.html): inside `.card-header` add
`.card-header-action > .card-header-btn` with `avatar-text avatar-xs bg-danger|bg-warning|bg-success`
links carrying `data-bs-toggle="remove" | "refresh" | "expand"`.

## List/table page pattern

Usage: one full-width card, `card-body p-0`, table inside `.table-responsive`. DataTables is
initialized on the table id by the page init script
(`$("#customerList").DataTable({pageLength:10,lengthMenu:[10,20,50,100,200,500]})` in
customers-init.min.js) — **pagination, page-length menu, search and info are generated by
DataTables at runtime; there is no hand-written pagination markup.** For server-rendered HTMX
tables, either keep DataTables client-side or emit Bootstrap `.pagination` yourself.
From customers.html.

```html
<div class="row">
    <div class="col-lg-12">
        <div class="card stretch stretch-full">
            <div class="card-body p-0">
                <div class="table-responsive">
                    <table class="table table-hover" id="customerList">
                        <thead>
                            <tr>
                                <th class="wd-30">
                                    <div class="btn-group mb-1">
                                        <div class="custom-control custom-checkbox ms-1">
                                            <input type="checkbox" class="custom-control-input" id="checkAllCustomer">
                                            <label class="custom-control-label" for="checkAllCustomer"></label>
                                        </div>
                                    </div>
                                </th>
                                <th>Customer</th>
                                <th>Email</th>
                                <th>Status</th>
                                <th class="text-end">Actions</th>
                            </tr>
                        </thead>
                        <tbody>
                            <tr class="single-item">
                                <td>
                                    <div class="item-checkbox ms-1">
                                        <div class="custom-control custom-checkbox">
                                            <input type="checkbox" class="custom-control-input checkbox" id="checkBox_1">
                                            <label class="custom-control-label" for="checkBox_1"></label>
                                        </div>
                                    </div>
                                </td>
                                <td>
                                    <!-- avatar + name cell -->
                                    <a href="customers-view.html" class="hstack gap-3">
                                        <div class="avatar-image avatar-md">
                                            <img src="assets/images/avatar/1.png" alt="" class="img-fluid">
                                        </div>
                                        <div>
                                            <span class="text-truncate-1-line">Alexandra Della</span>
                                        </div>
                                    </a>
                                </td>
                                <td><a href="apps-email.html">alex.della@outlook.com</a></td>
                                <td>
                                    <!-- status select rendered as a colored pill by select2-active.min.js -->
                                    <select class="form-control" data-select2-selector="status">
                                        <option value="success" data-bg="bg-success" selected>Active</option>
                                        <option value="warning" data-bg="bg-warning">Inactive</option>
                                        <option value="danger" data-bg="bg-danger">Declined</option>
                                    </select>
                                </td>
                                <td>
                                    <!-- row actions: eye icon + kebab dropdown, right-aligned -->
                                    <div class="hstack gap-2 justify-content-end">
                                        <a href="customers-view.html" class="avatar-text avatar-md">
                                            <i class="feather feather-eye"></i>
                                        </a>
                                        <div class="dropdown">
                                            <a href="javascript:void(0)" class="avatar-text avatar-md" data-bs-toggle="dropdown" data-bs-offset="0,21">
                                                <i class="feather feather-more-horizontal"></i>
                                            </a>
                                            <ul class="dropdown-menu">
                                                <li>
                                                    <a class="dropdown-item" href="javascript:void(0)">
                                                        <i class="feather feather-edit-3 me-3"></i>
                                                        <span>Edit</span>
                                                    </a>
                                                </li>
                                                <li class="dropdown-divider"></li>
                                                <li>
                                                    <a class="dropdown-item" href="javascript:void(0)">
                                                        <i class="feather feather-trash-2 me-3"></i>
                                                        <span>Delete</span>
                                                    </a>
                                                </li>
                                            </ul>
                                        </div>
                                    </div>
                                </td>
                            </tr>
                            <!-- repeat tr.single-item -->
                        </tbody>
                    </table>
                </div>
            </div>
        </div>
    </div>
</div>
```

Multi-tag cell variant (Group column): a multiple select rendered as colored tags by select2:

```html
<select class="form-select form-control max-select" data-select2-selector="tag" data-max-select2="tag" multiple>
    <option value="success" data-bg="bg-success">VIP</option>
    <option value="danger" data-bg="bg-danger" selected>Promotions</option>
</select>
```

## Create/edit form pattern

Usage: single full-width card whose header is a nav-tabs strip and whose panes are `.card-body`
sections. Fields use a horizontal 2-column layout: `col-lg-4` label / `col-lg-8` control, each
row is `row mb-4 align-items-center` (last row `mb-0`). Inputs get icon prefixes via
`.input-group > .input-group-text`. The template does not use `.is-invalid`/`.invalid-feedback`
markup statically — validation is jquery.validate on other pages; standard Bootstrap validation
classes work with theme.min.css. Save/cancel buttons are in the page header (see above).
From customers-create.html.

```html
<div class="row">
    <div class="col-lg-12">
        <div class="card border-top-0">
            <div class="card-header p-0">
                <!-- Nav tabs -->
                <ul class="nav nav-tabs flex-wrap w-100 text-center customers-nav-tabs" id="myTab" role="tablist">
                    <li class="nav-item flex-fill border-top" role="presentation">
                        <a href="javascript:void(0);" class="nav-link active" data-bs-toggle="tab" data-bs-target="#profileTab" role="tab">Profile</a>
                    </li>
                    <li class="nav-item flex-fill border-top" role="presentation">
                        <a href="javascript:void(0);" class="nav-link" data-bs-toggle="tab" data-bs-target="#passwordTab" role="tab">Password</a>
                    </li>
                </ul>
            </div>
            <div class="tab-content">
                <div class="tab-pane fade show active" id="profileTab" role="tabpanel">
                    <div class="card-body personal-info">
                        <!-- section heading with helper text and optional small action -->
                        <div class="mb-4 d-flex align-items-center justify-content-between">
                            <h5 class="fw-bold mb-0 me-4">
                                <span class="d-block mb-2">Personal Information:</span>
                                <span class="fs-12 fw-normal text-muted text-truncate-1-line">Following information is publicly displayed, be careful!</span>
                            </h5>
                            <a href="javascript:void(0);" class="btn btn-sm btn-light-brand">Add New</a>
                        </div>
                        <!-- text input with icon -->
                        <div class="row mb-4 align-items-center">
                            <div class="col-lg-4">
                                <label for="fullnameInput" class="fw-semibold">Name: </label>
                            </div>
                            <div class="col-lg-8">
                                <div class="input-group">
                                    <div class="input-group-text"><i class="feather-user"></i></div>
                                    <input type="text" class="form-control" id="fullnameInput" placeholder="Name">
                                </div>
                            </div>
                        </div>
                        <!-- textarea -->
                        <div class="row mb-4 align-items-center">
                            <div class="col-lg-4">
                                <label for="addressInput_2" class="fw-semibold">Address: </label>
                            </div>
                            <div class="col-lg-8">
                                <textarea class="form-control" id="addressInput_2" cols="30" rows="3" placeholder="Address"></textarea>
                            </div>
                        </div>
                        <!-- select (rendered by select2-active.min.js via data-select2-selector) -->
                        <div class="row mb-4 align-items-center">
                            <div class="col-lg-4">
                                <label class="fw-semibold">Status: </label>
                            </div>
                            <div class="col-lg-8">
                                <select class="form-control" data-select2-selector="status">
                                    <option value="success" data-bg="bg-success" selected>Active</option>
                                    <option value="warning" data-bg="bg-warning">Inactive</option>
                                    <option value="danger" data-bg="bg-danger">Declined</option>
                                </select>
                            </div>
                        </div>
                        <!-- other select flavors seen: data-select2-selector="country|state|city|tzone|currency|privacy",
                             data-select2-selector="language" multiple, and the "tag" multi-select (see table pattern).
                             Datepicker input: <input class="form-control" id="dateofBirth" placeholder="Pick date of birth">
                             (initialized by datepicker.min.js via the page init script). -->
                    </div>
                </div>
                <div class="tab-pane fade" id="passwordTab" role="tabpanel">
                    <div class="card-body pass-info">
                        <!-- same field rows -->
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>
```

Note: `card border-top-0` (not `stretch`) is used when the tabs strip is the card header;
each `.nav-item` carries `flex-fill border-top` so tabs span the full card width.

## Detail/view page pattern

Usage: two-column split — left `col-xxl-4 col-xl-6` stack of profile/info cards, right
`col-xxl-8 col-xl-6` tabbed card (same tabs-in-card-header pattern as forms). From
customers-view.html.

```html
<div class="row">
    <!-- LEFT: profile summary panel -->
    <div class="col-xxl-4 col-xl-6">
        <div class="card stretch stretch-full">
            <div class="card-body">
                <div class="mb-4 text-center">
                    <div class="wd-150 ht-150 mx-auto mb-3 position-relative">
                        <div class="avatar-image wd-150 ht-150 border border-5 border-gray-3">
                            <img src="assets/images/avatar/1.png" alt="" class="img-fluid">
                        </div>
                    </div>
                    <div class="mb-4">
                        <a href="javascript:void(0);" class="fs-14 fw-bold d-block">Alexandra Della</a>
                        <a href="javascript:void(0);" class="fs-12 fw-normal text-muted d-block">alex.della@outlook.com</a>
                    </div>
                </div>
                <!-- label/value rows with leading icon -->
                <ul class="list-unstyled mb-4">
                    <li class="hstack justify-content-between mb-4">
                        <span class="text-muted fw-medium hstack gap-3"><i class="feather-map-pin"></i>Location</span>
                        <a href="javascript:void(0);" class="float-end">California, USA</a>
                    </li>
                    <li class="hstack justify-content-between mb-0">
                        <span class="text-muted fw-medium hstack gap-3"><i class="feather-mail"></i>Email</span>
                        <a href="javascript:void(0);" class="float-end">alex.della@outlook.com</a>
                    </li>
                </ul>
                <div class="d-flex gap-2 text-center pt-4">
                    <a href="javascript:void(0);" class="w-50 btn btn-light-brand">
                        <i class="feather-trash-2 me-2"></i>
                        <span>Delete</span>
                    </a>
                    <a href="javascript:void(0);" class="w-50 btn btn-primary">
                        <i class="feather-edit me-2"></i>
                        <span>Edit Profile</span>
                    </a>
                </div>
            </div>
        </div>
        <!-- additional stacked cards (Social, Suggestions, ...) use the standard card pattern -->
    </div>
    <!-- RIGHT: tabbed detail panel -->
    <div class="col-xxl-8 col-xl-6">
        <div class="card border-top-0">
            <div class="card-header p-0">
                <ul class="nav nav-tabs flex-wrap w-100 text-center customers-nav-tabs" id="myTab" role="tablist">
                    <li class="nav-item flex-fill border-top" role="presentation">
                        <a href="javascript:void(0);" class="nav-link active" data-bs-toggle="tab" data-bs-target="#overviewTab" role="tab">Overview</a>
                    </li>
                    <li class="nav-item flex-fill border-top" role="presentation">
                        <a href="javascript:void(0);" class="nav-link" data-bs-toggle="tab" data-bs-target="#billingTab" role="tab">Billing</a>
                    </li>
                </ul>
            </div>
            <div class="tab-content">
                <div class="tab-pane fade show active p-4" id="overviewTab" role="tabpanel">
                    <div class="profile-details mb-5">
                        <div class="mb-4 d-flex align-items-center justify-content-between">
                            <h5 class="fw-bold mb-0">Profile Details:</h5>
                            <a href="javascript:void(0);" class="btn btn-sm btn-light-brand">Edit Profile</a>
                        </div>
                        <!-- definition rows: muted label / semibold value -->
                        <div class="row g-0 mb-4">
                            <div class="col-sm-6 text-muted">Full Name:</div>
                            <div class="col-sm-6 fw-semibold">Alexandra Della</div>
                        </div>
                        <div class="row g-0">
                            <div class="col-sm-6 text-muted">Email Address:</div>
                            <div class="col-sm-6 fw-semibold">alex.della@outlook.com</div>
                        </div>
                    </div>
                </div>
                <div class="tab-pane fade" id="billingTab" role="tabpanel">
                    <!-- pane content -->
                </div>
            </div>
        </div>
    </div>
</div>
```

## Buttons (variants actually used)

Usage: primary action `btn-primary`; secondary/tertiary `btn-light-brand` (the template's
"soft" brand button — used everywhere a plain secondary would be); icon-only square buttons
add `btn-icon`; small inline actions `btn btn-sm btn-light-brand`; danger `btn-danger`;
full-width large `btn btn-lg btn-primary w-100` (auth). Icon spacing is `me-2` inside a `<span>`-wrapped label.

```html
<a href="#" class="btn btn-primary"><i class="feather-plus me-2"></i><span>Create</span></a>
<a href="#" class="btn btn-light-brand"><i class="feather-layers me-2"></i><span>Save as Draft</span></a>
<a href="#" class="btn btn-icon btn-light-brand"><i class="feather-filter"></i></a>
<a href="#" class="btn btn-sm btn-light-brand">Edit Profile</a>
<a href="#" class="btn btn-md btn-light-brand"><i class="feather-filter me-2"></i><span>Filter</span></a>
<button type="submit" class="btn btn-lg btn-primary w-100">Login</button>
<!-- soft-tinted small button -->
<a href="#" class="btn btn-sm bg-soft-warning text-warning d-inline-block">Update Now</a>
<!-- circular icon "button" used for row actions and card kebabs -->
<a href="#" class="avatar-text avatar-md"><i class="feather feather-eye"></i></a>
```

## Badges / status indicators, avatars

Usage: soft badges pair `bg-soft-{color}` with `text-{color}`; solid badges use `bg-{color}`.
Header icon badges use `badge bg-danger nxl-h-badge`.

```html
<div class="badge bg-soft-success text-success">
    <i class="feather-arrow-up fs-10 me-1"></i>
    <span>36.85%</span>
</div>
<span class="badge bg-soft-danger text-danger">Declined</span>
<span class="badge bg-soft-success text-success ms-1">PRO</span>
<span class="badge bg-success nxl-h-badge">2</span>
```

Avatars — image type (`avatar-image`, round crop) and text/icon type (`avatar-text`, circular
outline). Sizes: `avatar-xs`, `avatar-sm`, `avatar-md`, `avatar-lg`, `avatar-xl`.

```html
<div class="avatar-image avatar-md">
    <img src="assets/images/avatar/1.png" alt="" class="img-fluid">
</div>
<div class="avatar-text avatar-lg bg-gray-200">
    <i class="feather-briefcase"></i>
</div>
<!-- overlapping avatar stack -->
<div class="img-group lh-0 ms-3">
    <!-- repeated a.avatar-image children -->
</div>
```

Status dot: `<i class="wd-10 ht-10 border border-2 border-gray-1 bg-success rounded-circle me-2"></i>`

## Tabs / nav-tabs pattern

Usage: on both create and view pages, tabs are the card header itself
(`.card-header.p-0 > ul.nav.nav-tabs.flex-wrap.w-100.text-center.customers-nav-tabs`), the card
gets `border-top-0`, and each `li.nav-item` carries `flex-fill border-top`. Panes:
`.tab-content > .tab-pane.fade[.show.active]` — pane content is either a `.card-body` div
(create page) or the pane itself padded with `p-4` (view page). Plain Bootstrap
`data-bs-toggle="tab"` / `data-bs-target="#paneId"` behavior; no custom JS needed.
See full markup in the form and detail patterns above.

## Empty state

There is no dedicated empty-state component in the template. The pattern actually used (header
timesheets dropdown, index.html) is a centered stack — big feather icon, muted text, small
primary button:

```html
<div class="d-flex justify-content-between align-items-center flex-column timesheets-body">
    <i class="feather-clock fs-1 mb-4"></i>
    <p class="text-muted">No started timers found yes!</p>
    <a href="javascript:void(0);" class="btn btn-sm btn-primary">Started Timer</a>
</div>
```

For empty tables/cards, reuse this shape inside a `.card-body text-center py-5`.

## Dismissible alert (bonus, used on view page)

```html
<div class="alert alert-dismissible mb-4 p-4 d-flex alert-soft-warning-message profile-overview-alert" role="alert">
    <div class="me-4 d-none d-md-block">
        <i class="feather feather-alert-triangle fs-1"></i>
    </div>
    <div>
        <p class="fw-bold mb-1 text-truncate-1-line">Your profile has not been updated yet!!!</p>
        <p class="fs-10 fw-medium text-uppercase text-truncate-1-line">Last Update: <strong>26 Dec, 2023</strong></p>
        <a href="javascript:void(0);" class="btn btn-sm bg-soft-warning text-warning d-inline-block">Update Now</a>
        <button type="button" class="btn-close" data-bs-dismiss="alert" aria-label="Close"></button>
    </div>
</div>
```
