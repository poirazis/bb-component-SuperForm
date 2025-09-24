# Super Form

An advanced form container component for Budibase applications with multi-step support, validation, and comprehensive form management.

## 🚀 Features

### Form Management

- **Create/Update Modes**: Support for both data creation and editing
- **Schema Integration**: Direct connection to Budibase data sources
- **Multi-step Forms**: Step-by-step form progression
- **Form Validation**: Comprehensive validation with visual feedback
- **State Tracking**: Form validity, dirtiness, and current step tracking

### Form Controls

- **Field Management**: Container for form field components
- **Action Integration**: Built-in form actions (validate, clear, step change)
- **Event Handling**: On change and reset event triggers
- **Dynamic Updates**: Real-time field value updates
- **Scroll Control**: Programmatic scrolling to form sections

### User Experience

- **Visual Feedback**: Clear validation states and error messages
- **Step Navigation**: Intuitive multi-step form navigation
- **Accessibility**: Full keyboard navigation and screen reader support
- **Responsive Design**: Adapts to different screen sizes
- **Loading States**: Visual feedback during form submission

### Advanced Features

- **Conditional Logic**: Dynamic field visibility and requirements
- **Data Binding**: Two-way data binding with data sources
- **Form Persistence**: Maintain form state across sessions
- **Bulk Operations**: Handle multiple record operations
- **Integration**: Full Budibase automation integration

### Styling & Layout

- **Flexible Sizing**: Shrink or grow container options
- **Custom Styling**: Padding, background, border, and shadow controls
- **Grid Layout**: Stretch alignment for form content
- **Theme Integration**: Consistent with Budibase design system

## 📝 Usage Instructions

### Basic Setup

1. Add the Super Form component to your screen
2. Select form type (Create or Update)
3. Connect to a data source schema
4. Add form field components as children
5. Configure validation and event handlers

### Advanced Configuration

- **Multi-step Setup**: Configure initial step and step navigation
- **Validation Rules**: Set up comprehensive form validation
- **Event Actions**: Attach workflows to form events
- **Styling**: Customize form appearance and layout

### Common Use Cases

- **Data Entry Forms**: User registration and profile forms
- **Survey Applications**: Multi-step survey and questionnaire forms
- **Admin Panels**: CRUD operations for data management
- **Application Workflows**: Step-by-step process forms
- **Configuration Forms**: Settings and preference forms

## 🔧 Configuration Options

| Setting           | Type    | Description                        |
| ----------------- | ------- | ---------------------------------- |
| Type              | Select  | Create or Update form mode         |
| Schema            | Schema  | Data source schema binding         |
| Disabled          | Boolean | Disable entire form                |
| Read Only         | Boolean | Read-only form mode                |
| Initial Form Step | Number  | Starting step for multi-step forms |
| Flex              | Select  | Container sizing (shrink/grow)     |

## 📋 Events

### On Change

Triggered when any form field value changes.

**Context:**

- `__value`: Complete form data object
- `__valid`: Overall form validity
- `__dirty`: Whether form has unsaved changes
- `__currentStep`: Current step number
- `__currentStepValid`: Current step validity

### On Reset

Triggered when form is reset to initial state.

## 🎯 Actions

- **ValidateForm**: Validate all form fields
- **ClearForm**: Reset form to initial state
- **ChangeFormStep**: Navigate to specific form step
- **UpdateFieldValue**: Update specific field value
- **ScrollTo**: Scroll to form section or field

## 🎨 Styling

Supports comprehensive styling options:

- **Layout**: Padding, margins, and spacing
- **Appearance**: Background, borders, and shadows
- **Size**: Flexible width and height controls
- **Theme**: Consistent with application design

## 🔍 Best Practices

- Use multi-step forms for complex data entry
- Implement proper validation for data integrity
- Provide clear feedback for form errors
- Consider mobile users for form layout
- Test form submission and error handling
- Use appropriate field grouping for clarity
