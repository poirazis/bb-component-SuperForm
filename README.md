# Super Form Pro

An enterprise-grade form component for Budibase applications with advanced features including multi-step workflows, sub-forms, data validation, and comprehensive form management.

## 🚀 Features

### Form Types & Modes

- **View/Create/Update Modes**: Full CRUD operations support
- **Sub-form Support**: Nested forms for relationships, JSON, and embedded documents
- **Multi-step Forms**: Step-by-step form progression with validation
- **Multi-row Forms**: Tabular data entry and editing
- **Schema Integration**: Direct connection to Budibase data sources

### Advanced Configuration

- **Column Layout**: Configurable form column count
- **Label Beautification**: Automatic label formatting
- **Permission Controls**: Edit and delete permissions
- **Disabled State**: Complete form disabling capability
- **Data Source Binding**: Flexible schema and data source connections

### Form Management

- **Validation System**: Comprehensive field and form validation
- **State Tracking**: Form validity, dirtiness, and current step monitoring
- **Action Integration**: Built-in form actions (validate, clear, update, scroll)
- **Event Handling**: Rich event system for form interactions
- **Context Provision**: Form data and state context for child components

### User Experience

- **Responsive Design**: Adapts to different screen sizes
- **Accessibility**: Full keyboard navigation and screen reader support
- **Visual Feedback**: Clear validation states and progress indication
- **Loading States**: Visual feedback during form operations
- **Error Handling**: Comprehensive error display and recovery

### Enterprise Features

- **Bulk Operations**: Handle multiple records and relationships
- **Data Refresh**: Automatic data source refreshing
- **Complex Relationships**: Support for nested and linked data structures
- **Audit Trail**: Form interaction and change tracking
- **Integration**: Full Budibase automation and workflow integration

## 📝 Usage Instructions

### Basic Setup

1. Add the Super Form Pro component to your screen
2. Select form type (View, Create, or Update)
3. Connect to a data source schema
4. Configure form layout and columns
5. Add form field components as children

### Advanced Configuration

- **Multi-step Setup**: Enable and configure step progression
- **Sub-forms**: Set up nested forms for complex data
- **Permissions**: Configure edit and delete capabilities
- **Validation**: Set up comprehensive form validation
- **Layout**: Customize column count and responsive behavior

### Common Use Cases

- **Complex Data Entry**: Multi-step registration and application forms
- **Administrative Panels**: CRUD operations for data management
- **Survey Systems**: Advanced survey and questionnaire forms
- **Configuration Interfaces**: System and user preference forms
- **Workflow Applications**: Process-driven form workflows

## 🔧 Configuration Options

| Setting         | Type    | Description                               |
| --------------- | ------- | ----------------------------------------- |
| Sub Form        | Boolean | Enable nested sub-form functionality      |
| Type            | Select  | Relationship/JSON/Embedded sub-form types |
| Action Type     | Select  | View/Create/Update form modes             |
| Schema          | Schema  | Data source schema binding                |
| Columns         | Number  | Form column count                         |
| Disabled        | Boolean | Disable entire form                       |
| Multi Step      | Boolean | Enable multi-step form progression        |
| Multi Row       | Boolean | Enable multi-row tabular editing          |
| Beautify Labels | Boolean | Automatic label formatting                |
| Can Edit        | Boolean | Allow editing in view mode                |
| Can Delete      | Boolean | Allow record deletion                     |

## 📋 Events

### On Change

Triggered when any form field value changes.

**Context:**

- `__value`: Complete form data object
- `__valid`: Overall form validity status
- `__dirty`: Whether form has unsaved changes
- `__currentStep`: Current step in multi-step forms
- `__currentStepValid`: Current step validation status

## 🎯 Actions

- **ValidateForm**: Validate all form fields
- **ClearForm**: Reset form to initial state
- **UpdateFieldValue**: Update specific field value
  - **Bulk Update Hack**: To update multiple fields at once, use field name `"_value"` with an object containing field-value pairs. The component will match keys (case-insensitively) to existing field names and update them accordingly. If the value is a JSON string, it will be parsed first. This provides a way to set the entire form state in a single action.
- **ScrollTo**: Scroll to form section or field
- **ChangeFormStep**: Navigate to specific form step
- **RefreshDatasource**: Refresh connected data sources

## 🎨 Styling

Supports comprehensive styling with:

- **Layout Control**: Column-based responsive layout
- **Visual States**: Validation and interaction states
- **Theme Integration**: Consistent with Budibase design
- **Custom CSS**: Advanced visual customization

## 🔍 Best Practices

- Use multi-step forms for complex workflows
- Configure appropriate permissions for security
- Implement validation for data integrity
- Consider mobile users for form layout
- Test sub-form functionality thoroughly
- Use appropriate column counts for readability
